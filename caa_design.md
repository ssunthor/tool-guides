# Cash Application Assistant (CAA) - Design Document

**Version:** 1.1.5
**Date:** 2025-11-02
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

* **BSI (Bank Statement Ingestor):** Provides new bank statement transactions via gRPC calls to BSI's ProcessBankStatement endpoint, which writes transaction data to the shared PostgreSQL database (bsi_transactions table). CAA monitors this table for new entries.
* **RMI (Remittance Mail Ingestor):** Provides processed remittance advice data stored in the shared PostgreSQL database (rmi_processing_results table). CAA queries this table during its rule execution.
* **CMC (CMD Connector):** Provides client lookup (via `GetClientByEmail`), open invoice data (via `GetOpenInvoices`), and master data for business units (via `GetBuBankInfo`) and invoice rules (via `GetInvoiceRules`). This data is populated into the CAA database via the `populate_bu_rules.py` script.
* **PostgreSQL Database:**
    * **Owns:** `cash_application_process` (state machine), `email_client_link` (linking cache), `rmi_open_invoice_match` (RMI matching cache), and master data tables (`business_units`, `bank_accounts`, `invoice_rules`).
    * **Reads:** `rmi_processing_results` (from RMI) and `bsi_transactions` / `bsi_statement_files` (from BSI).
* **Static Assets:** `oracle_remit_map.csv` is used by the `populate_bu_rules.py` script to enrich bank account data with Oracle Remittance Names.

### Outputs / Interactions:

* **PostgreSQL Database:** Updates the `cash_application_process` table with status changes, linked data (client ID, RMI ID), applied invoices, and processing notes. Writes results of background linking to `email_client_link` and RMI-invoice matching to `rmi_open_invoice_match`.
* **Redis:** Used for distributed locking (`linking_service`) to prevent concurrent runs of background tasks.
* **LLM Service (AI Factory):** Used by specific rules (e.g., `ApplyGlobalOrigAmtRule` for disambiguation) and the feedback analysis feature.
* **(Future) ORC (Oracle Connector):** Intended future integration point for posting successfully applied transactions to the Oracle ERP system. Currently Pending.
* **User Interface:** A React SPA served by the FastAPI backend provides monitoring, manual reprocessing triggers, rule management, and feedback submission.

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
    * **Logic:** Checks the associated `BsiTransaction` details (payer name, description, raw text, transaction type) against predefined keyword lists for `FEES_KEYWORDS`, `PAYROLL_KEYWORDS`, `INTERNAL_PAYER_NAMES`, and `INTERNAL_TRANSACTION_TYPES`.
    * **Conclusive Outputs:**
        * `ap_transfer`: If `amount < 0` (a debit).
        * `fees`, `payroll`, `internal_transfer`: If keywords match.
        * `skipped`: If general skip keywords (e.g., 'TAX') match or if `payer_name_raw` is missing/invalid.

2.  **`FindRmiRule`**
    * **Purpose:** To find a relevant Remittance Advice (`RmiProcessingResult`) record that likely corresponds to this bank transaction. This is the primary way to get a `sender_email` for client linking.
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
        1.  Checks the `email_client_link` table for a cached link for the RMI sender's email.
        2.  **Cache Hit:** If a valid link (`cmd_client_id` is present) is found, it applies it.
        3.  **Cache Fail (Recent):** If a *failed* lookup is cached and is recent (within 1 day), it trusts the failure.
        4.  **Cache Miss / Old Failed Cache:** Calls `cmc_connector.get_client_by_email` live.
            * If CMC returns multiple *active* clients (e.g., a `DOMAIN_MATCH`), it uses the RMI's `customer_name` and `fuzz.token_set_ratio` to find the best match.
            * If still ambiguous, it defaults to the client with the highest `contact_count`.
        5.  The live result (success or failure) is saved to the `email_client_link` table.
    * **State Change:**
        * On success: Sets `cmd_client_id`, `matched_client_name`, and changes status to `client_linked`.
        * On failure (e.g., no active clients found, CMC error): Changes status to `pending_client_invoice_link` (to fallback to BSI matching).

---

### State: `pending_client_invoice_link`

*Executed if no RMI was found, or if `MatchClientByEmailRule` failed.*

1.  **`MatchClientByBsiInvoiceRule`**
    * **Purpose:** To identify the client by finding invoice numbers mentioned in the BSI transaction details and looking them up in the `open_invoices` table.
    * **Logic:** Attempts to find invoice numbers using two strategies:
        1.  **Strategy 1 (Structured):** Uses the `BsiTransaction.extracted_invoice_numbers` list (data pre-parsed by BSI).
        2.  **Strategy 2 (Parsed):** If Strategy 1 fails, it uses `InvoiceParserHelper` to parse `raw_bank_text` and `reference_details` for invoice numbers matching BU-specific rules (prefix/length) from the `invoice_rules` table.
    * It queries `open_invoices` for the found document numbers. If all found, open invoices belong to a **single, unambiguous client group**, it links the process to the primary client from that group.
    * **State Change:**
        * On unique group match: Sets `matched_client_name` and changes status to `client_linked`.
        * On ambiguous match (multiple groups): Sets status to `hitl_needed` (Conclusive).
        * On failure (no invoices found or no unique group): Changes status to `pending_client_fuzzy_link`.

---

### State: `pending_client_fuzzy_link`

*Executed if RMI/Email and BSI Invoice linking both failed.*

1.  **`MatchClientByPayerPrefixRule`**
    * **Purpose:** To find a high-confidence client match by iteratively searching with an expanding prefix of the payer name, while handling common business abbreviations.
    * **Logic:**
        1.  Cleans the BSI `payer_name_raw` (e.g., "SYNAMEDIA LIMITED VSB...") and splits it into words.
        2.  It iterates from 1 word up to 5 words, building a search prefix (e.g., "SYNAMEDIA").
        3.  For each prefix, it checks for business synonyms in the *last word* (e.g., if the prefix is "SYNAMEDIA LIMITED", it generates a check list: `['SYNAMEDIA LIMITED%', 'SYNAMEDIA LTD%']`).
        4.  It queries `open_invoices` for `client_name` or `client_group` matching *all* prefixes in the check list.
        5.  **Evaluation:**
            * If the *total* number of unique clients found across all synonym-prefixes is **1**, the rule links that client and stops.
            * If the *total* number of clients found is **0**, the rule stops searching immediately (as a longer prefix will also find 0).
            * If the *total* number of clients found is **>1** (ambiguous), it logs the count and continues to the next, more specific prefix (e.g., "SYNAMEDIA LIMITED VSB%").
    * **State Change:** On a unique client name match, sets `matched_client_name` and status to `client_linked`. (Inconclusive on failure/ambiguity, allowing the next rule to run).

2.  **`MatchClientByFuzzyPayerRule`**
    * **Purpose:** To find a client using broad fuzzy name matching as a fallback.
    * **Logic:** Compares the full BSI `payer_name_raw` against *all* distinct `client_name` and `client_group` values from `open_invoices` using `fuzz.token_set_ratio`.
    * **State Change:**
        * On unique match (score >= 85): Sets `matched_client_name` and status to `client_linked`.
        * On ambiguous/no match: Changes status to `client_linked` *without* setting a client name. This forces the engine to proceed to the `client_linked` state and try global application rules.

---

### State: `client_linked`

*The main application stack. Runs if a client was linked at any previous step, or if fuzzy matching failed (to allow global rules). Rules run in this specific priority order.*

1.  **`ApplyRmiDirectRule`**
    * **Purpose:** To apply the payment if the linked RMI record's invoices can be matched 1:1 to open invoices for the *directly linked client*.
    * **Logic:**
        1.  Uses `RmiOpenInvoiceMatch` cache or raw `rmi_record.invoices`.
        2.  **BSI Conflict Safeguard:** Fails to HITL if BSI `extracted_invoice_numbers` exist and are *disjoint* (no overlap) with RMI invoices.
        3.  **Stale Cache Safeguard:** If no open invoices are found for an email-linked client, it triggers a *live* CMC call (`get_client_by_email`). If the live client ID is *different* from the cached one, it updates the process's `cmd_client_id` and *re-runs the entire `client_linked` rule stack* from the top.
        4.  **Validation:** Checks if all RMI invoices exist as open items for the `client_name` and if their `balance` matches the RMI `amount_paid` (within tolerance).
        5.  **Discrepancy Check:** Verifies that the `total_rmi_amount` (from lines or header) matches the `payment_amount` (within tolerance).
    * **Conclusive Outputs:** `applied_rmi_direct`, `hitl_needed` (mismatch).

2.  **`ApplyRmiGroupRule`**
    * **Purpose:** A fallback for `ApplyRmiDirectRule`. It checks if the RMI invoices match open items belonging to *any client* within the linked client's *group*.
    * **Logic:** Finds the client's `group_name` (from cache or `OpenInvoice` table). It then performs the same validation as `ApplyRmiDirectRule` but queries `open_invoices` by `client_group == group_name` instead of `client_name`.
    * **Conclusive Outputs:** `applied_rmi_group`, `hitl_needed` (if the same invoice number is found under multiple clients in that group).

3.  **`ApplyOrigAmtRule`**
    * **Purpose:** To apply a 1:1 payment that matches a single invoice's **original amount**. This is a common case for full, new payments.
    * **Logic:**
        1.  Finds all open invoices for the linked client where `original_amount` == `payment_amount` and `currency` matches.
        2.  **Crucial Check:** This rule *only* applies if the matched invoice's current `balance` *also* matches the `payment_amount`.
        3.  If `original_amount` matches but `balance` differs (e.g., a partial payment was already made), the rule is *inconclusive* and fails, preventing a misapplication.
        4.  If multiple invoices match both, it applies to the oldest (by `due_date`/`posting_date`).
    * **Conclusive Outputs:** `applied_amount_client`, `applied_amount_client_oldest`.

4.  **`ApplyBalanceMatchRule`**
    * **Purpose:** To apply a 1:1 payment that matches a single invoice's **current balance**. This is common for paying off a remaining, partially-paid invoice.
    * **Logic:**
        1.  Finds all open invoices for the linked client where `balance` == `payment_amount` and `currency` matches.
        2.  If one or more matches are found, it applies the payment to the oldest one (by `due_date`/`posting_date`).
    * **Conclusive Outputs:** `applied_balance_client`, `applied_balance_client_oldest`.

5.  **`ApplyNettedBsiRule`**
    * **Purpose:** To apply a payment that matches the sum of a *group* of invoices/memos (>= 2) that were extracted directly by BSI.
    * **Logic:**
        1.  Requires `BsiTransaction.extracted_invoice_numbers` to have 2 or more items.
        2.  Identifies if it's an FX payment (BSI `instructed_amount` is valid and currency differs from `payment_currency`). Sets `target_amount` and `target_currency` accordingly.
        3.  Finds all matching open items in the client's *group* and `target_currency`.
        4.  Calculates `sum(balance)` and `sum(original_amount)` of the found items.
        5.  **Full Match:** If *all* BSI invoices were found, it checks if `sum(balance)` OR `sum(original_amount)` matches the `target_amount` (within 0.01).
        6.  **Partial Match:** If *not all* BSI invoices were found (e.g., some were missing/paid), it checks if the sum (`balance` or `original_amount`) of the *remaining* items matches the `target_amount` (within a looser 0.10 tolerance).
    * **Conclusive Outputs:** `applied_bsi_netted`, `applied_bsi_netted_orig_amt`, `applied_bsi_partial_net`, `applied_bsi_netted_fx`, etc.

6.  **`ApplyBsiDailyGroupRule`**
    * **Purpose:** To apply a payment that matches the total sum of *all* open items for a client on a *single day*.
    * **Logic:**
        1.  **Anchor:** If BSI extracted an invoice, it finds that invoice's `posting_date` and uses it as the `target_date`.
        2.  **No Anchor:** If BSI extracted no invoices, this rule iterates through *every* open posting date for the linked client to find a `sum` match.
    * For the target date(s), it checks if `sum(balance)` OR `sum(original_amount)` matches the `payment_amount` (within 0.10 tolerance).
    * **Conclusive Outputs:** `applied_bsi_daily_group_orig_amt`, `applied_bsi_daily_group_bal`. If no group sum matches *and* no anchor was used, it checks the *total* client balance and can set `hitl_client_total_mismatch`.

7.  **`ApplyNetFeeRule`**
    * **Purpose:** To apply a 1:1 payment where a bank fee was deducted (e.g., payment $990, fee $10, invoice $1000).
    * **Logic:**
        1.  Finds a *separate* debit (`D`) transaction in the `bsi_transactions` table that shares the *same `bank_transaction_id`* and contains a fee keyword.
        2.  If exactly one fee is found, it calculates `net_amount = payment_amount + fee_amount`.
        3.  It then queries `open_invoices` for the client to find a single item where `balance` == `net_amount`.
    * **Conclusive Outputs:** `applied_amount_client_net_fee`, `hitl_needed` (ambiguous).

8.  **`ApplyInstructedAmtRule`**
    * **Purpose:** To apply a 1:1 FX payment using BSI's structured `instructed_amount` field.
    * **Logic:**
        1.  Checks if `BsiTransaction.instructed_amount` and `instructed_currency` are present and different from the payment currency.
        2.  Queries `open_invoices` for the client where `original_amount` == `instructed_amount` and `currency` == `instructed_currency`.
        3.  Applies to the oldest if multiple match.
    * **Conclusive Outputs:** `applied_bsi_instructed_amount`.

9.  **`ApplyBsiRawFxRule`**
    * **Purpose:** To apply an FX payment by parsing raw, unstructured text when `ApplyInstructedAmtRule` fails.
    * **Logic:**
        1.  Only runs if `ApplyInstructedAmtRule` was inconclusive (signaled by a "Skipped" note).
        2.  Parses `raw_bank_text` for regex patterns: `SETT-AMT: [CUR] [AMT]` and `RATE: [RATE]`.
        3.  Requires BSI-extracted invoices (`extracted_invoice_numbers`) to be present.
        4.  **Math Check:** Validates that `(Settlement Amount * Rate)` matches the `payment_amount` (within 0.02 tolerance).
        5.  **Application Check:** Validates that the `Settlement Amount` matches the `sum(balance)` or `sum(original_amount)` of the BSI-extracted invoices (which must be in the `Settlement Currency`).
    * **Conclusive Outputs:** `applied_bsi_raw_fx`, `hitl_needed`, `error`.

10. **`ApplyGlobalOrigAmtRule`**
    * **Purpose:** A global 1:1 fallback. To apply a payment that matches a *single* invoice's `original_amount` *globally*, regardless of client linkage.
    * **Logic:**
        1.  Queries `open_invoices` globally (no client filter) where `original_amount` == `payment_amount`.
        2.  **1 Match:** Links the client from that invoice and applies the payment.
        3.  **>1 Match (Same Client):** Applies to the oldest.
        4.  **>1 Match (Diff Clients):** Uses an **LLM** (via `OllamaClient` and `rule6_disambiguator` prompt) to compare the BSI `payer_name_raw` against the candidate client names. If the LLM returns a single, confident match, it links that client and applies the payment.
    * **Conclusive Outputs:** `applied_amount_global`, `hitl_needed` (ambiguous, LLM failed).

11. **`ApplyPartialBsiRefRule`**
    * **Purpose:** A global group fallback. To apply a payment by reconstructing partial invoice numbers from BSI text.
    * **Logic:**
        1.  Uses `InvoiceParserHelper` to find *only* numeric strings (e.g., '12345') in the BSI raw text that match the *length* of known invoice rules (e.g., a 5-digit rule).
        2.  Reconstructs potential `doc_no` using the rule's prefix (e.g., `G12345`).
        3.  Queries for these reconstructed `doc_no`s *globally*.
        4.  If all found items belong to a **single, unambiguous client group** and their `sum(balance)` matches the `payment_amount`, it links the client and applies the payment.
    * **Conclusive Outputs:** `applied_bsi_ref`, `hitl_needed` (ambiguous).

---

### State: `pending_hitl_diagnosis`

*Executed if the `client_linked` or `pending_client_fuzzy_link` states fail to produce a match.*

1.  **`DiagnoseRmiFailureRule`**
    * **Purpose:** To provide a specific HITL reason for RMI-based failures.
    * **Logic:** Checks the `rmi_open_invoice_match` table. If the RMI match *failed* (e.g., `Amount Mismatch`, `Invoice(s) Not Found`), it sets the process status to `hitl_rmi_mismatch`. It copies the *failed* invoice list (enriching it with balance/original amount from `open_invoices`) into the `applied_invoices` field for UI review.
    * **Conclusive Output:** `hitl_rmi_mismatch`.

2.  **`DiagnoseBsiFailureRule`**
    * **Purpose:** To provide specific HITL reason for BSI-based group failures.
    * **Logic:** Checks the process `notes` for failure text logged by `ApplyBsiDailyGroupRule` or `ApplyNettedBsiRule` (e.g., "did not match payment").
    * **Conclusive Outputs:** `hitl_bsi_daily_group_mismatch`, `hitl_bsi_amount_mismatch`, `hitl_partial_bsi_match`.

3.  **`DiagnoseClientNotFoundRule`**
    * **Purpose:** To provide specific HITL reason if no client was ever linked.
    * **Logic:** Checks if `cmd_client_id` and `matched_client_name` are both `None`.
    * **Conclusive Output:** `hitl_client_not_found`.

* **(Fallback)** If no diagnostic rule applies, the `RuleEngine`'s fallback logic sets the status to `hitl_needed` (or a rule-specific HITL like `hitl_client_total_mismatch`).

---

## 5. Supporting Services

### Linking Service (`linking_service.py`)

* **Purpose:** To proactively and periodically link RMI sender emails to CMD clients, creating a cache (`email_client_link`) used by the main orchestrator (`MatchClientByEmailRule`).
* **Trigger:** Runs periodically in the background (controlled by `caa_main.py`).
* **Process:**
    1.  Uses Redis for distributed locking to ensure only one instance runs at a time.
    2.  Queries `rmi_processing_results` for distinct sender emails not present in `email_client_link`.
    3.  For each new email, it calls `cmc_connector.get_client_by_email`.
    4.  It applies fuzzy matching logic (using RMI `customer_name` vs. CMC client results) to select the best client match or determine failure/ambiguity.
    5.  The result (success or failure) is inserted/updated in the `email_client_link` table.
* **Resilience:** Includes delays and handles rate limiting/timeouts from CMC.

### RMI Invoice Matching Service (`invoice_matching_service.py`)

* **Purpose:** To proactively match RMI records against open invoices, populating the `rmi_open_invoice_match` table. This provides immediate match status in the Remittance UI and supplies cached data for the main orchestrator's `ApplyRmiDirectRule` and `DiagnoseRmiFailureRule`.
* **Trigger:** Runs periodically in the background (controlled by `caa_main.py`).
* **Process:**
    1.  Queries for RMI records that are eligible for matching (e.g., RMI status is `SUCCESS...`, match status is `Pending`, or a failed match is old enough to retry).
    2.  For each RMI record, it executes its own dedicated **RMI Rule Engine** (`core/rmi_rule_engine.py`).
    3.  The RMI Rule Engine runs a specific chain of rules (from `core/rmi_match_rules/`) in priority order:

        * **`MatchClientInvoicesRule`:** (Highest priority) Checks if the invoices listed in the RMI (e.g., `['123', '124']`) exist as open items for the *linked client* and if the amounts match. It performs a two-pass check:
            * **Pass 1 (Inv#):** Matches by exact `doc_no`. It checks both `balance` and `original_amount` for a match, resulting in `Matched (Balance)` or `Matched (Orig Amt)`.
            * **Pass 2 (Amount):** For any RMI invoices *not* found in Pass 1, it queries for open invoices for that client with a matching `original_amount`. It picks the oldest if multiple are found.

        * **`MatchGlobalInvoiceRule`:** (Fallback) If `MatchClientInvoicesRule` fails (e.g., invoices not found for the *linked client*), this rule re-queries for the RMI `doc_no`s *globally* (without a client filter). If it finds an unambiguous match, it records it.

        * **`MatchClientTotalBalanceRule`:** (No RMI Invoices) If the RMI contains no invoice list, this rule checks if the `total_payment_amount` from the RMI *exactly* matches a single open invoice `balance` for the linked client.

        * **`MatchClientGroupedBalanceRule`:** (No RMI Invoices) If `MatchClientTotalBalanceRule` fails, this rule checks if the `total_payment_amount` matches the `sum(balance)` of an entire *daily posting group* for the linked client.

        * **`MatchGlobalOriginalAmountRule`:** (No RMI Invoices) As a final fallback, this rule checks if the `total_payment_amount` matches a single open invoice's `original_amount` *globally* (any client, any currency).

    4.  The final status (e.g., `Matched`, `Partial Match`, `Amount Mismatch`, `Invoice(s) Not Found`) is upserted into the `rmi_open_invoice_match` table.

---

## 6. User Interface (React SPA)

The CAA UI is a Single Page Application built with React, Vite, and Material UI, served by the FastAPI backend. It provides:

* **Monitoring:** Views for Cash Application Processes, Remittances (RMI), Bank Statements (BSI), and Open Invoices, with filtering, sorting, and detailed drill-downs.
* **Intervention:** Buttons to trigger reprocessing for individual items, bulk reprocessing based on filters, and daily processing triggers for bank statements.
* **Configuration:** A dedicated page (`/rules`) for managing Business Units, their associated Bank Accounts, and Invoice/Credit Memo parsing rules.
* **Feedback:** A system for users to submit feedback on specific RMI processing results or general issues, including AI-driven analysis and response generation.
* **Visualization:** A dashboard providing high-level statistics on RMI processing, invoice matching, and cash application statuses.

---

## 7. Future Enhancements & TODO Plan

* **ORC Integration:** Develop the Oracle Connector (ORC) module and integrate it with CAA. Add a final step to the orchestrator for successfully applied transactions (status in `SUCCESSFUL_APPLICATION_STATUSES`) to call an ORC gRPC endpoint (e.g., `PostCashApplication`) to record the application in the Oracle ERP system. This will involve updating the process status further (e.g., `posted_to_oracle`).
* **Refine Rule Logic:** Continuously evaluate the effectiveness of the existing rules based on `hitl_needed` cases and user feedback. Adjust thresholds (e.g., fuzzy score), add new rules, or modify existing logic as needed.
* **UI Improvements:** Add more detailed reporting, potentially user-assignable tasks for HITL items, and richer visualizations.