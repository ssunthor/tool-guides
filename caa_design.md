# Cash Application Assistant (CAA) - Design Document

**Version:** 1.5.4
**Date:** 2025-12-16
**Author(s):** Martin Kecka, Gemini AI
**Category:** Assistant (A)
**Acronym:** CAA

---

## 1. Introduction & Purpose

The **Cash Application Assistant (CAA)** is an AI-enhanced orchestration module designed to automate the process of matching incoming bank statement transactions (payments) to corresponding remittance advices and open invoices. Its primary goal is to determine how a received payment should be applied, reducing manual effort and improving the speed and accuracy of cash application.

CAA operates by processing individual bank transactions ingested by the Bank Statement Ingestor (BSI). For each transaction, it executes a sequence of predefined rules to:

* Identify and Skip Non-Payment Transactions: Filter out items like bank fees, payroll, or internal transfers.
* Link to Supporting Data: Find related Remittance Advice details processed by the Remittance Mail Ingestor (RMI).
* Identify the Client: Determine the correct client entity associated with the payment using data from RMI, BSI details, and the CMD Connector (CMC).
* Match to Open Invoices: Compare payment details and remittance information against open invoice data (sourced via CMC) to determine the specific items being paid.
* Determine Final Application Status: Assign a status indicating successful automatic application, the need for human review (HITL), or an error state.

The system provides a user interface for monitoring the status of each transaction, reviewing the logic applied, and intervening manually when necessary.

---

## 2. Architecture & Integration

* **Service Type:** (A) Assistant Module with a user-facing React/FastAPI UI and background processing capabilities.
* **Core Function:** Orchestrates the cash application process based on rules, managing the state of each incoming bank transaction.

### Data Sources / Inputs:

* **BSI (Bank Statement Ingestor):** Provides new bank statement transactions via shared database tables (`bsi_transactions`, `bsi_statement_files`). The CAA background poller (`bsi_processing_service`) monitors these tables for `unprocessed` records.
* **RMI (Remittance Mail Ingestor):** Provides processed remittance advice data stored in the shared PostgreSQL database (`rmi_processing_results` table). CAA queries this table during its rule execution.
* **CMC (CMD Connector):** Provides client lookup (via `GetClientByEmail`), open invoice data (via `GetOpenInvoices`), and master data.
* **PostgreSQL Database:**
    * **State Machine:** `cash_application_process` (central orchestration state).
    * **Staging Layer:** `oracle_receipt_queue` (buffers transactions before ERP posting).
    * **Learning/Cache:** `email_client_link` (email-to-client), `payer_client_links` (manual payer name overrides), and `rmi_open_invoice_match` (RMI match cache).
    * **Master Data (Split Schema):**
        * `physical_bank_accounts`: Represents the actual bank ledger (owned by an Oracle Entity).
        * `cmd_business_units`: Represents the invoicing entity (CMD).
        * `bu_bank_assignments` & `cmd_account_links`: Configures the many-to-many relationship between physical accounts and logical business units, allowing shared service center models.
* **Static Assets:** `oracle_remit_map.csv` is used by population scripts to enrich bank account data.

### Outputs / Interactions:

* **PostgreSQL Database:** Updates the `cash_application_process` table with status changes, linked data (client ID, RMI ID), applied invoices, and processing notes. Manages the Oracle Staging Queue.
* **Redis:** Used for distributed locking in background services (`linking_service`, `invoice_matching_service`, `bsi_processing_service`) to prevent concurrent runs.
* **LLM Service (AI Factory):** Used by specific rules (e.g., `ApplyGlobalOrigAmtRule`, `MatchClientByOnBehalfOfRule`) and the feedback analysis feature.
* **ORC (Oracle Connector):** gRPC connector used to sync receipt status and post "Ready" receipts from the Staging Queue to the Oracle ERP (`CreateStandardReceipt`).
* **User Interface:** A React SPA served by the FastAPI backend provides monitoring, the Oracle Reconciliation Workbench, manual rule configuration, and HITL resolution.

---

## 3. Core Functionality & Process Flow ⚙️

CAA operates primarily as a state machine driven by the `status` field in the `cash_application_process` table.

### Initiation

A new process starts when either:

* A new `BsiTransaction` record appears in the database (typically ingested via the BSI module).
* A user manually triggers processing or reprocessing for a specific date or process ID via the API (`/api/cash-application/process-day/{date_str}` or `/api/cash-application/process/{process_id}/reprocess-step`).
* A `CashApplicationProcess` record is created with `status = 'new'`.

### Orchestration (`application_orchestrator.py`)

The `application_orchestrator.process_next_step` function is a thin wrapper that retrieves the `CashApplicationProcess` and passes it to the `core.rule_engine.RuleEngine`.

The **RuleEngine** (`core/rule_engine.py`) operates as a state machine:

1.  It checks the process's current `status` (e.g., `new`, `pending_client_email`, `client_linked`).
2.  It executes a specific list of rules associated with that status (see Section 4 for the rule list).
3.  Rules run in a defined priority order.
4.  If a rule is **conclusive** (sets a terminal status like `applied_...`, `hitl_...`, `skipped`, or `error`), the engine stops and commits the transaction.
5.  If a rule is **inconclusive** but *changes the state* (e.g., `new` -> `pending_client_email`), the engine immediately re-evaluates the rules for the *new* state.
6.  If all rules for a state are inconclusive and the state doesn't change, it triggers a **Fallback**.

**Fallback & Diagnosis:**

* If application rules (for `client_linked` or `pending_client_fuzzy_link`) fail to find a match, the status is set to `pending_hitl_diagnosis`.
* The engine then runs a final set of `hitl_rules` (e.g., `DiagnoseRmiFailureRule`, `DiagnoseBsiFailureRule`, `DiagnoseClientNotFoundRule`).
* These rules analyze the process state (e.g., checking RMI match data, process notes) to set a specific, descriptive `hitl_...` status (e.g., `hitl_rmi_mismatch`, `hitl_bsi_daily_group_mismatch`, `hitl_client_not_found`).
* This provides a clear reason for the fallout in the UI, replacing a generic "HITL" status.
* If no diagnostic rule applies, the status defaults to `hitl_needed`.

### Background Tasks

* **Periodic Linking (`linking_service.py`):** Runs periodically to find new sender emails in RMI results that haven't been linked to clients yet. It calls CMC and updates the `email_client_link` table. This provides cached data used by `MatchClientByEmailRule`.
* **Periodic RMI Invoice Matching (`invoice_matching_service.py`):** This service now uses its own dedicated **RMI Rule Engine** (from `core/rmi_rule_engine.py`) to proactively match RMI records against open invoices, storing the outcome in `rmi_open_invoice_match`. This data is primarily used by the Cash App process rules (e.g., `ApplyRmiDirectRule`) but also provides insights on the Remittance page UI.

---

## 4. The Orchestration Rules (Detailed Logic)

The core logic resides in the `RuleEngine`, which executes rules in a specific order based on the process's current state. The rule names (e.g., `CheckSkipConditionsRule`) correspond to the class names in `core/orchestrator_rules/`.

---

### State: `new`

*Executed when a process is first created.*

1.  **`CheckSkipConditionsRule`**
    * **Purpose:** To identify and filter out transactions that are clearly not standard customer payments early in the process.
    * **Logic:** Dynamically fetches active rules from the `skip_rules` database table. It compares the transaction text, payer name, or transaction code against configured values using Exact, Contains, Starts With, or Regex matching.
    * **Conclusive Outputs:** `ap_transfer` (if debit), `fees`, `payroll`, `internal_transfer`, `tax`, `skipped`.

2.  **`FindRmiRule`**
    * **Purpose:** To find a relevant Remittance Advice (`RmiProcessingResult`) record that likely corresponds to this bank transaction.
    * **Logic:** Queries `RmiProcessingResult` for records matching the `payment_amount` (exact), `payment_currency`, and a date window (-7/+2 days). It then **validates** potential matches by:
        1.  Checking payer name similarity: `fuzz.token_set_ratio(bsi_payer, rmi_customer) >= 70`.
        2.  Checking for invoice conflicts: If BSI *and* RMI both extracted invoice numbers, the two sets *must not* be disjoint (i.e., they must share at least one invoice).
    * **State Change:**
        * On validated match: Sets `rmi_processing_id` and changes status to `pending_client_email`.
        * On failure (no match or validation fail): Changes status to `pending_client_invoice_link`.

---

### State: `pending_client_email`

*Executed if `FindRmiRule` successfully linked an RMI record.*

1.  **`MatchClientByEmailRule`**
    * **Purpose:** To identify the CMD client using the sender email from the linked RMI record.
    * **Logic:**
        1.  Checks the `email_client_link` table for a cached link.
        2.  **Cache Hit:** If a valid link exists, applies it. If a *failed* lookup is cached and recent (<24h), it trusts the failure.
        3.  **Cache Miss:** Calls `cmc_connector.get_client_by_email` live.
            * If multiple active clients return (e.g., `DOMAIN_MATCH`), it uses `rmi_customer_name` fuzzy matching to disambiguate.
            * If still ambiguous, defaults to the client with the highest `contact_count`.
        4.  Saves the result to `email_client_link`.
    * **State Change:**
        * On success: Sets `cmd_client_id`, `matched_client_name`, status -> `client_linked`.
        * On failure: Status -> `pending_client_invoice_link`.

---

### State: `pending_client_invoice_link`

*Executed if no RMI was found, or if `MatchClientByEmailRule` failed.*

1.  **`MatchClientByBsiInvoiceRule`**
    * **Purpose:** To identify the client by finding invoice numbers mentioned in the BSI transaction.
    * **Logic:** Uses two strategies:
        1.  **Strategy 1 (Structured):** Uses `BsiTransaction.extracted_invoice_numbers` (pre-parsed).
        2.  **Strategy 2 (Parsed):** If Strategy 1 fails, it uses `InvoiceParserHelper` to parse `raw_bank_text` using BU-specific regex rules.
    * **Action:** Queries `open_invoices` for found numbers. If all found invoices belong to a **single, unambiguous client group**, it links the primary client from that group.
    * **State Change:**
        * On unique group: Status -> `client_linked`.
        * On ambiguous group: Status -> `hitl_needed` (Conclusive).
        * On failure: Status -> `pending_client_fuzzy_link`.

---

### State: `pending_client_fuzzy_link`

*Executed if RMI/Email and BSI Invoice linking both failed. This state attempts increasingly "fuzzy" methods to identify the payer.*

1.  **`MatchClientByManualLinkRule`**
    * **Purpose:** Checks for a persistent manual override created by a human operator in the UI.
    * **Logic:** Queries the `payer_client_links` table for an exact match on the normalized BSI Payer Name.
    * **State Change:** On match: Status -> `client_linked`.

2.  **`MatchClientByOnBehalfOfRule`**
    * **Purpose:** Extracts the true client name from narratives like "PAYMENT ON BEHALF OF [CLIENT]".
    * **Logic:**
        1.  Uses Regex to extract the candidate name string after "ON BEHALF OF".
        2.  Performs Fuzzy Matching against the `open_invoices` client list.
        3.  **LLM Disambiguation:** If the fuzzy match is ambiguous (60-90%), it sends the candidate string and the top database matches to an LLM to select the correct legal entity name.
    * **State Change:** On match: Status -> `client_linked`.

3.  **`MatchClientByPayerPrefixRule`**
    * **Purpose:** Finds a client by iteratively searching with expanding prefixes of the payer name.
    * **Logic:**
        1.  Iterates from 1 to 5 words of the payer name.
        2.  Handles business suffixes (e.g., searching "SYNAMEDIA LIMITED" also checks "SYNAMEDIA LTD").
        3.  **Recency Tie-Breaker:** If a prefix matches multiple clients (Ambiguous), it checks which client had the most recent invoice activity (posting date). If one client is significantly more active (>180 days newer), it wins.
    * **State Change:**
        * Unique Match: Status -> `client_linked`.
        * Group Match (Ambiguous Client): Sets `matched_group_name`, Status -> `pending_client_fuzzy_link` (allows `MatchClientByGroupAndBuRule` to run).

4.  **`MatchClientByGroupAndBuRule`**
    * **Purpose:** Resolves "Ambiguous Group" scenarios (e.g., 'Leadec Group') found by previous rules using the bank account context.
    * **Logic:**
        1.  Identifies the **Receiving Bank Account's** linked Business Units (via `physical_bank_accounts` -> `cmd_account_links`).
        2.  Filters the candidates within the identified Client Group to include *only* those that belong to one of the linked BUs.
    * **State Change:** If filtering results in a single client: Status -> `client_linked`.

5.  **`MatchClientByFuzzyPayerRule`**
    * **Purpose:** Final fallback using broad fuzzy string matching.
    * **Logic:** Compares the full `payer_name_raw` against *all* distinct `client_name` and `client_group` values from `open_invoices` using `fuzz.token_set_ratio`.
    * **State Change:**
        * Unique match (>85%): Status -> `client_linked`.
        * Ambiguous/No match: Status -> `client_linked` (without setting a client name). This forces the engine to proceed to the `client_linked` state to try global application rules (e.g., Global Amount Match).

---

### State: `client_linked`

*The main application stack. Runs if a client was linked OR if fuzzy matching failed (to allow global rules). Rules run in this specific priority order.*

1.  **`ApplyRmiDirectRule`**
    * **Purpose:** Applies payment using the invoices listed in the linked RMI record.
    * **Logic:**
        1.  Checks if BSI invoices conflict with RMI invoices (Safeguard).
        2.  Validates that RMI invoices exist as open items for the *linked client* and amounts match.
        3.  **Total Sum Fallback:** If individual line-item matching fails (e.g., bad OCR on one invoice number), the rule calculates the sum of *all found* open invoices referenced in the RMI. If this sum matches the `payment_amount` (within tolerance), it ignores the bad line data and applies the payment.
        4.  **Stale Cache Safeguard:** If no open invoices are found for an email-linked client, it triggers a *live* CMC call (`get_client_by_email`). If the live client ID is *different* from the cached one, it updates the process's `cmd_client_id` and re-runs the stack.
    * **Conclusive Outputs:** `applied_rmi_direct`, `hitl_needed` (mismatch).

2.  **`ApplyRmiGroupRule`**
    * **Purpose:** Fallback for `ApplyRmiDirectRule` that searches across the entire Client Group.
    * **Logic:** Finds the client's `group_name` (from cache or `OpenInvoice` table). It then performs the same validation as `ApplyRmiDirectRule` (including Total Sum Fallback) but queries `open_invoices` by `client_group == group_name`.
    * **Conclusive Outputs:** `applied_rmi_group`, `hitl_needed` (if the same invoice number is found under multiple clients in that group).

3.  **`ApplyOrigAmtRule`**
    * **Purpose:** 1:1 match on Original Amount.
    * **Logic:**
        1.  Finds all open invoices for the linked client where `original_amount` == `payment_amount` and `currency` matches.
        2.  **Crucial Check:** This rule *only* applies if the matched invoice's current `balance` *also* matches the `payment_amount`.
        3.  If `original_amount` matches but `balance` differs (e.g., a partial payment was already made), the rule is *inconclusive* and fails, preventing a misapplication.
        4.  If multiple invoices match both, it applies to the oldest (by `due_date`/`posting_date`).
    * **Conclusive Outputs:** `applied_amount_client`, `applied_amount_client_oldest`.

4.  **`ApplyBalanceMatchRule`**
    * **Purpose:** 1:1 match on Current Balance.
    * **Logic:**
        1.  Finds all open invoices for the linked client where `balance` == `payment_amount` and `currency` matches.
        2.  If one or more matches are found, it applies the payment to the oldest one (by `due_date`/`posting_date`).
    * **Conclusive Outputs:** `applied_balance_client`, `applied_balance_client_oldest`.

5.  **`ApplyPaidAmountMatchRule`**
    * **Purpose:** Applies payment that matches the *difference* between Original Amount and Balance.
    * **Logic:** Queries open invoices where `(OriginalAmount - Balance) == PaymentAmount`. This handles cases where the customer pays the "paid portion" or reconciles a specific ledger state.
    * **Conclusive Outputs:** `applied_paid_amt_match`.

6.  **`ApplyNettedBsiRule`**
    * **Purpose:** To apply a payment that matches the sum of a *group* of invoices/memos (>= 2) that were extracted directly by BSI.
    * **Logic:**
        1.  Requires `BsiTransaction.extracted_invoice_numbers` to have 2 or more items.
        2.  Identifies if it's an FX payment (BSI `instructed_amount` is valid and currency differs from `payment_currency`). Sets `target_amount` and `target_currency` accordingly.
        3.  Finds all matching open items in the client's *group* and `target_currency`.
        4.  Calculates `sum(balance)` and `sum(original_amount)` of the found items.
        5.  **Full Match:** If *all* BSI invoices were found, it checks if `sum(balance)` OR `sum(original_amount)` matches the `target_amount` (within 0.01).
        6.  **Partial Match:** If *not all* BSI invoices were found (e.g., some were missing/paid), it checks if the sum (`balance` or `original_amount`) of the *remaining* items matches the `target_amount` (within a looser 0.10 tolerance).
    * **Conclusive Outputs:** `applied_bsi_netted`, `applied_bsi_netted_orig_amt`, `applied_bsi_partial_net`, `applied_bsi_netted_fx`.

7.  **`ApplyBsiDailyGroupRule`**
    * **Purpose:** To apply a payment that matches the total sum of *all* open items for a client on a *single day*.
    * **Logic:**
        1.  **Anchor:** If BSI extracted an invoice, it finds that invoice's `posting_date` and uses it as the `target_date`.
        2.  **Remainder Logic:** If the extracted invoice is only a partial match, searches for *other* open invoices on that same date to fill the remaining balance.
        3.  **No Anchor:** If BSI extracted no invoices, this rule iterates through *every* open posting date for the linked client to find a `sum` match.
        4.  Checks if `sum(balance)` OR `sum(original_amount)` matches the `payment_amount` (within 0.10 tolerance).
    * **Conclusive Outputs:** `applied_bsi_daily_group_orig_amt`, `applied_bsi_daily_group_bal`. If no group sum matches *and* no anchor was used, it checks the *total* client balance and can set `hitl_client_total_mismatch`.

8.  **`ApplyNetFeeRule`**
    * **Purpose:** To apply a 1:1 payment where a bank fee was deducted (e.g., payment $990, fee $10, invoice $1000).
    * **Logic:**
        1.  Finds a *separate* debit (`D`) transaction in the `bsi_transactions` table that shares the *same `bank_reference`* and contains a fee keyword.
        2.  If exactly one fee is found, it calculates `net_amount = payment_amount + fee_amount`.
        3.  It then queries `open_invoices` for the client to find a single item where `balance` == `net_amount`.
    * **Conclusive Outputs:** `applied_amount_client_net_fee`, `hitl_needed` (ambiguous).

9.  **`ApplyInstructedAmtRule`**
    * **Purpose:** To apply a 1:1 FX payment using BSI's structured `instructed_amount` field.
    * **Logic:**
        1.  Checks if `BsiTransaction.instructed_amount` and `instructed_currency` are present and different from the payment currency.
        2.  Queries `open_invoices` for the client where `original_amount` == `instructed_amount` and `currency` == `instructed_currency`.
        3.  Applies to the oldest if multiple match.
    * **Conclusive Outputs:** `applied_bsi_instructed_amount`.

10. **`ApplyBsiRawFxRule`**
    * **Purpose:** To apply an FX payment by parsing raw, unstructured text when `ApplyInstructedAmtRule` fails.
    * **Logic:**
        1.  Parses `raw_bank_text` for regex patterns: `SETT-AMT: [CUR] [AMT]` and `RATE: [RATE]`.
        2.  Requires BSI-extracted invoices (`extracted_invoice_numbers`) to be present.
        3.  **Math Check:** Validates that `(Settlement Amount * Rate)` matches the `payment_amount` (within 0.02 tolerance).
        4.  **Application Check:** Validates that the `Settlement Amount` matches the `sum(balance)` or `sum(original_amount)` of the BSI-extracted invoices (which must be in the `Settlement Currency`).
    * **Conclusive Outputs:** `applied_bsi_raw_fx`, `hitl_needed`, `error`.

11. **`ApplyGlobalOrigAmtRule`**
    * **Purpose:** Global 1:1 Fallback (Client not linked).
    * **Logic:**
        1.  Queries globally for `original_amount == payment_amount`.
        2.  **1 Match:** Links the client from that invoice and applies the payment.
        3.  **>1 Match (Same Client):** Applies to the oldest.
        4.  **>1 Match (Diff Clients):** Uses an **LLM** (via `OllamaClient` and `rule6_disambiguator` prompt) to compare the BSI Payer Name against the candidate client names. If the LLM returns a single, confident match, it links that client and applies the payment.
    * **Conclusive Outputs:** `applied_amount_global`, `hitl_needed` (ambiguous, LLM failed).

12. **`ApplyPartialBsiRefRule`**
    * **Purpose:** Global Group Fallback.
    * **Logic:**
        1.  Uses `InvoiceParserHelper` to find *only* numeric strings (e.g., '12345') in the BSI raw text that match the *length* of known invoice rules (e.g., a 5-digit rule).
        2.  Reconstructs potential `doc_no` using the rule's prefix (e.g., `G12345`).
        3.  Queries for these reconstructed `doc_no`s *globally*.
        4.  If all found items belong to a **single, unambiguous client group** and their `sum(balance)` matches the `payment_amount`, it links the client and applies the payment.
    * **Conclusive Outputs:** `applied_bsi_ref`, `hitl_needed` (ambiguous).

---

### State: `pending_hitl_diagnosis`

*Executed if the `client_linked` or `pending_client_fuzzy_link` states fail to produce a match. This state runs diagnostic rules to determine the specific reason for the failure or to catch edge cases like closed invoices.*

1.  **`DiagnoseClosedInvoiceRule`**
    * **Purpose:** To check if the failure is due to the invoices already being paid/closed in the ERP (which would cause standard "Open Invoice" rules to fail).
    * **Logic:**
        1.  **RMI Check:** Checks the `rmi_open_invoice_match` table. If the RMI pre-match status is `Matched (Closed)`, it applies the payment.
        2.  **Local BSI Check:** Checks if BSI-extracted invoice numbers exist in the local database with `status = 'CLOSED'`.
        3.  **Live ERP Fallback:** If BSI invoices are missing locally, it triggers a **Live ERP Check** via the CMC Connector. If the invoices exist in the ERP as closed/zero-balance, it syncs them to the local database and applies the payment.
    * **Conclusive Output:** `applied_closed_invoice` (Success - sets status to `closed` in Oracle Staging).

2.  **`DiagnoseRmiFailureRule`**
    * **Purpose:** To provide a specific HITL reason for RMI-based failures.
    * **Logic:** Checks the `rmi_open_invoice_match` table. If the RMI match *failed* (e.g., `Amount Mismatch`, `Invoice(s) Not Found`), it sets the process status to `hitl_rmi_mismatch`. It copies the *failed* invoice list (enriching it with balance/original amount from `open_invoices`) into the `applied_invoices` field for UI review.
    * **Conclusive Output:** `hitl_rmi_mismatch`.

3.  **`DiagnoseBsiFailureRule`**
    * **Purpose:** To provide specific HITL reason for BSI-based group failures.
    * **Logic:** Checks the process `notes` for failure text logged by `ApplyBsiDailyGroupRule` or `ApplyNettedBsiRule` (e.g., "did not match payment").
    * **Conclusive Outputs:** `hitl_bsi_daily_group_mismatch`, `hitl_bsi_amount_mismatch`, `hitl_partial_bsi_match`.

4.  **`DiagnoseClientNotFoundRule`**
    * **Purpose:** To provide specific HITL reason if no client was ever linked.
    * **Logic:** Checks if `cmd_client_id` and `matched_client_name` are both `None`.
    * **Conclusive Output:** `hitl_client_not_found`.

* **(Fallback)** If no diagnostic rule applies, the `RuleEngine`'s fallback logic sets the status to `hitl_needed` (or a rule-specific HITL like `hitl_client_total_mismatch`).
---

## 5. Supporting Services

### BSI Processing Service (`bsi_processing_service.py`)
* **Purpose:** Background poller that automates the ingestion-to-orchestration pipeline.
* **Trigger:** Runs every 30 minutes (configurable).
* **Process:**
    1.  **Pre-computation:** Triggers `open_invoices` sync to ensure Closed/Paid statuses are up to date in the local cache.
    2.  **Refresh:** Refreshes recent failed RMI matches (last 7 days) to catch up on data latency (e.g., invoices that opened *after* the email arrived). This prevents premature "Amount Mismatch" errors.
    3.  **Ingestion:** Queries `bsi_transactions` for `unprocessed` items and initiates `application_orchestrator.start_application_process`.

### Linking Service (`linking_service.py`)
* **Purpose:** To proactively and periodically link RMI sender emails to CMD clients, creating a cache (`email_client_link`) used by the main orchestrator (`MatchClientByEmailRule`).
* **Trigger:** Runs periodically in the background (controlled by `caa_main.py`).
* **Process:**
    1.  Uses Redis for distributed locking to ensure only one instance runs at a time.
    2.  Queries `rmi_processing_results` for distinct sender emails not present in `email_client_link`.
    3.  For each new email, calls `cmc_connector.get_client_by_email`.
    4.  Applies fuzzy matching logic (using RMI `customer_name` vs. CMC client results) to select the best client match or determine failure/ambiguity.
    5.  Saves the result (success or failure) to the `email_client_link` table.
* **Learning:** Now supports "Learning" from manual overrides. If a user manually links a process in the UI, the `MatchClientByManualLinkRule` runs, and the linking service updates its cache to respect that link logic for future emails.

### Oracle Staging Service (`oracle_staging_service.py`)
* **Purpose:** Manages the `oracle_receipt_queue` database table—the "Reconciliation Workbench."
* **Functionality:**
    * **Balance Reconciliation:** Pulls Opening and Closing balances from BSI file headers and compares them against the sum of staged transactions (`Posted + Ready + HITL + Excluded`) to calculate variance.
    * **Sequence Generation:** Generates Oracle Receipt Numbers (e.g., `DB9141.12162025.01`). Ensures that excluded transactions (Fees, Internal) do not consume sequence numbers, maintaining a gap-free audit trail.
    * **Bi-Directional Sync:**
        * **Push:** Bulk-posts "Ready" receipts to Oracle via the ORC Connector (`CreateStandardReceipt`).
        * **Sync:** Checks receipt status from Oracle to update local records (idempotency check) and handle reversals.
    * **Audit Logging:** Captures all changes (e.g., user edits to amounts, applied invoices) and logs the diff for AI feedback analysis.

### RMI Invoice Matching Service (`invoice_matching_service.py`)
* **Purpose:** To proactively match RMI records against open invoices, populating the `rmi_open_invoice_match` table. This provides immediate match status in the Remittance UI and supplies cached data for the main orchestrator's `ApplyRmiDirectRule` and `DiagnoseRmiFailureRule`.
* **Trigger:** Runs periodically in the background (controlled by `caa_main.py`).
* **Process:**
    1.  Queries for RMI records eligible for matching (e.g., status is `SUCCESS...`, match status is `Pending`, or failed match is old enough to retry).
    2.  Executes the dedicated **RMI Rule Engine** (`core/rmi_rule_engine.py`).
    3.  The RMI Rule Engine runs a specific chain of rules (from `core/rmi_match_rules/`) in priority order:

        * **`MatchClientInvoicesRule`:** (Highest priority) Checks if the invoices listed in the RMI (e.g., `['123', '124']`) exist as open items for the *linked client* and if the amounts match.
            * **Pass 1 (Inv#):** Matches by exact `doc_no`. Checks both `balance` and `original_amount`.
            * **Pass 2 (Amount):** For missing invoices, queries for open invoices matching `original_amount`.
            * **Live Fallback:** If invoices are missing locally, calls `MatchLiveInvoiceRule` to check the live ERP.

        * **`MatchLiveInvoiceRule`:** (New) If local matching fails, queries the live ERP via CMC. If invoices are found (even if `CLOSED` or zero-balance), syncs them to the local DB and retries the match.

        * **`MatchGlobalInvoiceRule`:** (Fallback) If `MatchClientInvoicesRule` fails, this rule re-queries for the RMI `doc_no`s *globally* (without a client filter). If it finds an unambiguous match, it records it.

        * **`MatchClientTotalBalanceRule`:** (No RMI Invoices) If the RMI contains no invoice list, this rule checks if the `total_payment_amount` from the RMI *exactly* matches a single open invoice `balance` for the linked client.

        * **`MatchClientGroupedBalanceRule`:** (No RMI Invoices) If `MatchClientTotalBalanceRule` fails, this rule checks if the `total_payment_amount` matches the `sum(balance)` of an entire *daily posting group* for the linked client.

        * **`MatchGlobalOriginalAmountRule`:** (Final Fallback) This rule checks if the `total_payment_amount` matches a single open invoice's `original_amount` *globally* (any client, any currency).

    4.  The final status (e.g., `Matched`, `Partial Match`, `Amount Mismatch`, `Matched (Closed)`) is upserted into the `rmi_open_invoice_match` table.
    5.  **Retry Logic:** Implements smart exponential backoff. Errors retry up to 3 times (1-hr interval), while verified "Not Found" records retry once (24-hr interval).

## 6. Manual Intervention & Learning System

The system is designed to improve over time via Human-in-the-Loop (HITL) interactions.

1.  **Manual Client Linking:**
    * Users can manually link a raw Payer Name to a CMD Client ID via the UI dialog.
    * **Persistence:** This creates a record in `payer_client_links`.
    * **Automation:** The `MatchClientByManualLinkRule` runs *first* in the linking phase (State: `pending_client_fuzzy_link`), ensuring the system "remembers" this correction for all future payments from that payer.

2.  **Categorization & Exclusion:**
    * Users can mark transactions as "Tax", "Fees", or "Internal" in the Oracle Staging view.
    * **Rule Creation:** The UI allows converting these one-off actions into permanent `SkipRule` database entries (e.g., "Always skip payer 'Internal Transfer'").

3.  **Auto-Feedback Loop:**
    * When a user edits a receipt amount or invoice application in the Staging area, `feedback_processor.py` captures the "diff".
    * It uses an LLM to analyze *why* the user made the change (e.g., "User added $15 fee") and logs it for system tuning.

## 7. Oracle Staging & Reconciliation Workbench

The **Oracle Staging Viewer** acts as a final firewall before data enters the ERP. It provides a robust interface for Finance users to review, correct, and commit cash applications.

### Key Features

1.  **Balance Reconciliation Logic:**
    * The view pulls **Opening Balance** and **Closing Balance** directly from the BSI Statement File (`bsi_statement_files` table).
    * It calculates a **Projected Closing Balance** by summing the amounts of all transactions in the staging queue (`Posted`, `Ready`, `HITL`, `Excluded`).
    * **Variance Check:** It compares the `Projected` vs `Actual` Closing Balance.
        * **Green:** Zero Variance (The queue perfectly matches the bank statement).
        * **Red:** Variance detected (Funds are missing from the queue or amounts are incorrect).

2.  **Sequence Logic & Gap Prevention:**
    * Oracle Receipt Numbers are generated automatically using the pattern `{BankPrefix}{Last4}.{MMDDYYYY}.{Sequence}` (e.g., `DB9141.11272025.01`).
    * **Exclusion Logic:** Transactions marked as `Excluded`, `Fees`, or `Internal Transfer` do **not** consume a sequence number. This ensures that the final posted receipts in Oracle have a continuous, gap-free sequence (e.g., `.01`, `.02`, `.03`), which is critical for audit trails.

3.  **Smart Status Grouping:**
    * **Active Items:** Receipts that require action (`Ready`, `HITL`, `Error`) are grouped at the top.
    * **Excluded Items:** Non-posting items are collapsed in a separate "Excluded" section to reduce clutter while maintaining audit visibility.
    * **Closed Items:** Transactions matched to already-closed invoices are flagged but kept visible for reference.

4.  **Bi-Directional Sync:**
    * **Push:** Bulk-posts "Ready" receipts to Oracle via the ORC Connector (`CreateStandardReceipt`).
    * **Sync:** Users can manually sync the status of a specific receipt *from* Oracle. The system checks if the receipt already exists (Idempotency) using the generated Receipt Number and updates the local status to `POSTED` if a match is found, preventing duplicates.

5.  **Edit Capabilities:**
    * Users can modify **Receipt Number**, **Comments**, **Currency**, and **Applied Invoices** via a modal dialog.
    * Changes trigger an **Auto-Feedback Loop** (see Section 6), logging the "diff" for AI analysis.

---

## 8. User Interface (React SPA)

The CAA UI is a Single Page Application built with React, Vite, and Material UI, served by the FastAPI backend. It provides:

* **Monitoring:** Views for Cash Application Processes, Remittances (RMI), Bank Statements (BSI), and Open Invoices, with filtering, sorting, and detailed drill-downs.
* **Intervention:** Buttons to trigger reprocessing for individual items, bulk reprocessing based on filters, and daily processing triggers for bank statements.
* **Configuration:** A dedicated page (`/rules`) for managing Business Units, their associated Bank Accounts, and Invoice/Credit Memo parsing rules.
* **Feedback:** A system for users to submit feedback on specific RMI processing results or general issues, including AI-driven analysis and response generation.
* **Visualization:** A dashboard providing high-level statistics on RMI processing, invoice matching, and cash application statuses.

---

## 9. Future Enhancements & TODO Plan

### 9.1 Functional Roadmap
* **User Task Assignment:** Update the data model to allow assigning specific HITL transactions to individual users (e.g., `assigned_to_user_id`), creating personalized "My Work" views for Finance operators.
* **Direct Posting Mode:** Implement a "Fast Track" configuration that allows high-confidence matches (e.g., 100% Exact Match on Invoice & Amount) to bypass the Oracle Staging Queue and post directly to ERP, further reducing manual review volume.
* **Archive Function:** Once Oracle is fully processed we should "archive" all related items from our database

### 9.2 Documentation & AI Context Optimization
* **Glossary of Concepts:** Create a strict definition list distinguishing *BSI Payer*, *CMD Client*, and *Oracle Customer* to prevent LLM hallucinations during feedback analysis.
* **Failure Mode Documentation:** Explicitly document the *inverse* conditions for each rule (e.g., "Why did `ApplyNettedBsiRule` fail?") to help the AI explain the "Why" to users.
* **Ambiguity Logic Maps:** Document the specific tie-breakers used in client linking (Recency vs. Context vs. LLM) so the AI can explain why a specific client was chosen over another.
* **Status Translation Layer:** Create a mapping table that translates internal technical statuses (e.g., `hitl_bsi_amount_mismatch`) into user-friendly explanations for the AI to use in chat responses.
* **Configuration Traceability:** Explicitly map system behaviors to their controlling database tables (`skip_rules`, `invoice_rules`) to help the AI suggest configuration fixes.