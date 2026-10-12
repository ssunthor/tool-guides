# Cash Application Assistant (CAA) - Design Document

**Version:** 2.4.5
**Date:** 2026-09-17
**Author:** Martin Kecka & Gemini AI
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

* **BSI (Bank Statement Ingestor):** Provides new bank statement transactions via shared database tables (`bsi_transactions`, `bsi_statement_files`). The CAA background poller (`bsi_processing_service`) monitors these tables for `unprocessed` records. Also includes Addenda Processing (BofA US) for deep metadata like Check/Wire/ACH lockbox details.
* **RMI (Remittance Mail Ingestor):** Provides processed remittance advice data stored in the shared PostgreSQL database (`rmi_processing_results` table). CAA queries this table during its rule execution.
* **CMC (CMD Connector):** Provides client lookup (via `GetClientByEmail`), open invoice data (via `GetOpenInvoices`), and master data.
* **PostgreSQL Database:**
    * **State Machine:** `cash_application_process` (central orchestration state).
    * **Staging Layer:** `oracle_receipt_queue` (buffers transactions before ERP posting, supports 1:N splitting).
    * **Audit & History:** `transaction_audit_logs` (dedicated event-sourced audit trail capturing structural system events, splits, manual interventions, and operator metrics).
    * **Learning/Cache:** `email_client_link` (email-to-client), `payer_client_links` (manual payer name overrides), `rmi_open_invoice_match` (RMI match cache), and `feedback_logs` (capturing `context_data` JSONB before/after state snapshots).
    * **Master Data (Split Schema):**
        * `physical_bank_accounts`: Represents the actual bank ledger (owned by an Oracle Entity).
        * `cmd_business_units`: Represents the invoicing entity (CMD).
        * `bu_bank_assignments` & `cmd_account_links`: Configures the many-to-many relationship between physical accounts and logical business units, allowing shared service center models.
        * `aggregators`: Manages third-party payment aggregator definitions (e.g., Bill.com, Cass Info) dynamically.
* **Static Assets:** `oracle_remit_map.csv` is used by population scripts to enrich bank account data.

### Outputs / Interactions:

* **PostgreSQL Database:** Updates the `cash_application_process` table with status changes, linked data, applied invoices, and processing notes. Manages the Oracle Staging Queue.
* **Redis:** Used for distributed locking in background services (`linking_service`, `invoice_matching_service`, `bsi_processing_service`, scheduled syncs) to prevent concurrent runs and API DDOSing. Also utilizes Sorted Sets (ZSETs) for managing asynchronous, event-driven polling tasks (e.g., Intercompany Bridge generation via `cross_bu_worker.py`).
* **LLM Service (AI Factory):** Served via `vLLMClient` (OpenAI-compatible) powered by Gemma-4-31B-it. Used by specific rules (e.g., `ApplyGlobalOrigAmtRule`, `MatchClientByOnBehalfOfRule`) and the feedback/resolution analysis feature.
* **ORC (Oracle Connector):** gRPC connector used to sync receipt status, apply credit memos (`ApplyCreditMemo`), unapply credit memos, fetch live ERP data, and post receipts via SOAP Atomic requests (`CreateStandardReceipt` & `ApplyStandardReceipt`) followed by REST verification.
* **User Interface:** A React SPA served by the FastAPI backend provides monitoring, the Oracle Reconciliation Workbench, manual rule configuration, and HITL resolution.
* **AMU Observability:** Integrates the SERA gRPC standard `SeraAppService` for centralized Application Management Utility (AMU) health reporting (e.g., PostgreSQL, Redis metrics).
* **Configuration:** Adheres to the SERA ecosystem `v1-universal` standard for the `settings.yaml` file, ensuring a unified configuration structure across the application (including database, services, and `llm_engines`).

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

* **Smart BSI Processing Poller (`bsi_processing_service.py`):** Uses an adaptive scheduling algorithm (High Frequency during Peak UTC hours, Low Frequency off-hours). It validates extracted invoice numbers against the ERP before processing to prevent data loss during CMC downtimes.
* **Scheduled Open Invoice Sync (`open_invoices.py`):** Uses APScheduler with Redis locking to trigger full background refreshes at fixed UTC intervals to align with Oracle sync cycles.
* **Periodic Linking (`linking_service.py`):** Runs periodically to find new sender emails in RMI results that haven't been linked to clients yet. It calls CMC and updates the `email_client_link` table.
* **Periodic RMI Invoice Matching (`invoice_matching_service.py`):** Proactively matches RMI records against open invoices using a dedicated Rule Engine. This process also facilitates **RMI-to-Email Learning**, mapping emails to clients automatically upon a hard invoice match.

---

## 4. The Orchestration Rules (Detailed Logic)

The core logic resides in the `RuleEngine`, which executes rules in a specific order based on the process's current state. The rule names (e.g., `CheckSkipConditionsRule`) correspond to the class names in `core/orchestrator_rules/`.

---

### State: `new`

*Executed when a process is first created.*

1.  **`CheckSkipConditionsRule`**
    * **Purpose:** To identify and filter out transactions that are clearly not standard customer payments early in the process.
    * **Logic:** Dynamically fetches active rules from the `skip_rules` database table (with a 60-second in-memory cache). Utilizes a flexible JSONB multi-criteria AND logic (e.g., matching Payer Name AND Transaction Text simultaneously) using Exact Match, Contains, Starts With, or Regex matching. Also inherently checks debit indicators (e.g., 'D', 'RC') to automatically flag outflows.
    * **Conclusive Outputs:** `ap_transfer` (if debit), `fees`, `payroll`, `internal_transfer`, `tax`, `credit_card_payment`, `interests`, `skipped`.

2.  **`HandleLockboxSummaryRule`**
    * **Purpose:** Prevents double-posting from Lockbox files containing both individual checks and summary deposits.
    * **Logic:** Flags transactions explicitly marked `is_summary=True` by the BSI addenda parser (e.g., BAI2 Code 115 Total Deposits).
    * **Conclusive Output:** `hitl_lockbox_summary`.

3.  **`FindRmiRule`**
    * **Purpose:** To find a relevant Remittance Advice (`RmiProcessingResult`) record that likely corresponds to this bank transaction.
    * **Logic:** 1. **Duplicate Guard:** Actively checks if a candidate RMI is already linked to another active Cash Application Process to prevent duplicate applications.
        2. **Targeted Aggregator Search:** If the BSI Payer Name maps to a known aggregator in the `aggregators` table, it explicitly searches for RMIs from that aggregator's email, bypassing strict currency checks.
        3. Queries `RmiProcessingResult` for records matching the `payment_amount` (exact, or gross amount if BSI charges are present), `payment_currency`, and a date window (-7/+2 days). 
        4. Features a **3-Tier Tie-Breaker Strategy** for ambiguous/multiple RMI candidate scenarios:
            1.  **Amazon Payment Ref:** Matches the BSI `PAYMENT-NUM` string against email body content.
            2.  **BSI Fallback/Overlap:** Checks if BSI text contains an invoice number unique to one RMI candidate. If candidates contain identical invoice data (duplicates), the most recent one is auto-selected.
            3.  **Invoice Owner Look-Ahead:** Extracts invoice numbers from RMI candidates, looks up their owner in the DB, and fuzzy matches that owner against the bank Payer Name (`fuzz.token_set_ratio >= 70`).
        * Also checks for invoice conflicts: If BSI *and* RMI both extracted invoice numbers, the two sets *must not* be disjoint. If RMI remains ambiguous but BSI has valid invoices, it skips RMI linking to prevent unnecessary HITL stops.
    * **State Change:**
        * Validated match -> `pending_client_email`.
        * Failure/Ambiguous -> `pending_client_invoice_link` or `hitl_multiple_rmi_candidates`.

---

### State: `pending_client_email`

*Executed if `FindRmiRule` successfully linked an RMI record.*

1.  **`MatchClientByEmailRule`**
    * **Purpose:** To identify the CMD client using the sender email from the linked RMI record.
    * **Logic:**
        1.  **Dynamic Aggregator Check:** Actively detects Aggregators based on the database `aggregators` table. If the email matches a known aggregator, it enforces an Aggregator Mode link (`cmd_client_id = None`, `is_aggregator = True`) to force invoice-level, multi-customer matching later.
        2.  Checks the `email_client_link` table for a cached link. If a *failed* lookup is cached and recent (<24h), it trusts the failure.
        3.  **Cache Miss:** Calls `cmc_connector.get_client_by_email` live. Evaluates fuzzy thresholds between the RMI Customer Name and the active CMD client names. If multiple return and fuzzy scoring is ambiguous, defaults to the highest `contact_count`.
    * **State Change:** Success -> `client_linked`. Failure -> `pending_client_invoice_link`.

---

### State: `pending_client_invoice_link`

*Executed if no RMI was found, or if `MatchClientByEmailRule` failed.*

1.  **`MatchClientByBsiInvoiceRule`**
    * **Purpose:** To identify the client by finding invoice numbers mentioned in the BSI transaction.
    * **Logic:**
        1.  **Strategy 1 (Structured):** Uses `BsiTransaction.extracted_invoice_numbers` (pre-parsed).
        2.  **Strategy 2 (Parsed):** Uses `InvoiceParserHelper` to parse `raw_bank_text` using BU-specific regex rules.
        * Queries `open_invoices` for found numbers (where balance != 0). Links the client if all invoices belong to a **single, unambiguous client group**. If linked, immediately stores the matched items into `applied_invoices` to optimize downstream processing.
        * **Sticky Bank Template Bypass:** Intelligently checks if conflicting BSI-extracted invoices are already closed in the ERP. If so, it safely ignores the stale bank text and allows the system to rely on current RMI data.
    * **State Change:** Unique group -> `client_linked`. Ambiguous group -> `hitl_needed`. Failure -> `pending_client_fuzzy_link`.

---

### State: `pending_client_fuzzy_link`

*Executed if RMI/Email and BSI Invoice linking both failed. This state attempts increasingly "fuzzy" methods to identify the payer.*

1.  **`MatchClientByManualLinkRule`**
    * **Purpose:** Persistent Payer Learning. Checks for a persistent manual override created by a human operator in the UI.
    * **Logic:** Queries the `payer_client_links` table for an exact match on the normalized BSI Payer Name. This ensures manual HITL resolutions are "remembered".
    * **State Change:** On match: Status -> `client_linked`.

2.  **`MatchClientByHistoryRule`**
    * **Purpose:** Organizational Memory.
    * **Logic:** Looks up historical successful processes (`STATUS_APPLIED_...`) to see if this specific Payer Name was previously linked to a client and successfully applied. If found, replicates the link.
    * **State Change:** On match: Status -> `client_linked`.

3.  **`MatchClientByOnBehalfOfRule`**
    * **Purpose:** Extracts the true client name from narratives like "PAYMENT ON BEHALF OF [CLIENT]".
    * **Logic:**
        1.  Uses Regex to extract the candidate name string after "ON BEHALF OF".
        2.  Performs Fuzzy Matching against the `open_invoices` client list.
        3.  **LLM Disambiguation:** If the fuzzy match is ambiguous (60-90%), it sends the candidate string and the top database matches to an LLM (`vLLMClient`) to logically select the correct legal entity name.
    * **State Change:** On match: Status -> `client_linked`.

4.  **`MatchClientByPayerPrefixRule`**
    * **Purpose:** Finds a client by iteratively searching with expanding prefixes of the payer name.
    * **Logic:**
        1.  Iterates from 1 to 5 words of the payer name using Strict Forward Accumulation.
        2.  Handles business suffixes and dot-aware variants (e.g., searching "B.V." also checks "BV", "LIMITED" checks "LTD").
        3.  **Recency Tie-Breaker:** If a prefix matches multiple clients (Ambiguous), it checks which client had the most recent invoice activity (posting date). If one client is significantly more active (>180 days newer), it wins.
    * **State Change:**
        * Unique Match: Status -> `client_linked`.
        * Group Match (Ambiguous Client): Sets `matched_group_name`, Status -> `pending_client_fuzzy_link` (allows `MatchClientByGroupAndBuRule` to run).

5.  **`MatchClientByGroupAndBuRule`**
    * **Purpose:** Resolves "Ambiguous Group" scenarios (e.g., 'Leadec Group') found by previous rules using the bank account context.
    * **Logic:**
        1.  Identifies the **Receiving Bank Account's** linked Business Units (via `physical_bank_accounts` -> `cmd_account_links` or `oracle_links`).
        2.  Filters the candidates within the identified Client Group to include *only* those that belong to one of the linked BUs.
    * **State Change:** If filtering results in a single client: Status -> `client_linked`.

6.  **`ApplyBsiGroupWithCreditMemoRule` (Pre-check)**
    * **Purpose:** Attempts an early group-level application if a credit memo is involved.
    * **State Change:** If successful: Status -> `applied_bsi_group_with_cm`.

7.  **`MatchClientByFuzzyPayerRule`**
    * **Purpose:** Final fallback using broad fuzzy string matching.
    * **Logic:** Compares the full `payer_name_raw` against *all* distinct `client_name` and `client_group` values from `open_invoices` using `fuzz.token_set_ratio`.
    * **State Change:**
        * Unique match (>85%): Status -> `client_linked`.
        * Ambiguous/No match: Status -> `client_linked` (without setting a client name). This intentionally forces the engine to proceed to the `client_linked` state to try global application rules (e.g., Global Amount Match).

---

### State: `client_linked`

*The main application stack. Runs if a client was linked OR if fuzzy matching failed (to allow global rules). Rules run in this STRICT priority order.*

1.  **`ApplyMultiCustomerRmiRule`**
    * **Purpose:** Handles payments from Aggregators (e.g., Cass, Bill.com).
    * **Logic:** If the RMI email matches a known aggregator, it bypasses client-level isolation and performs a **Global Search** for every invoice listed in the RMI. It correctly splits the single bank transaction into multiple applications across different customers. Supports **Partial Match Tolerance** (applies valid ones, flags the missing ones as `hitl_rmi_multi_customer_partial`).
    * **Conclusive Outputs:** `applied_rmi_multi_customer`, `hitl_rmi_multi_customer_partial`.

2.  **`ApplyRmiClosedMatchRule`**
    * **Purpose:** Strict Prioritization of Closed matches.
    * **Logic:** Checks if the background `invoice_matching_service` already verified the RMI invoices are `CLOSED` in the ERP. If so, accepts the result immediately. This prevents subsequent "Amount Match" heuristics from blindly grabbing unrelated open items just to satisfy the payment amount.
    * **Conclusive Outputs:** `applied_closed_invoice`.

3.  **`ApplyRmiDirectRule`**
    * **Purpose:** Applies payment using the invoices listed in the linked RMI record.
    * **Logic:**
        1.  Checks if BSI invoices conflict with RMI invoices (Safeguard).
        2.  Validates that RMI invoices exist as open items for the *linked client* and amounts match perfectly.
        3.  **Total Sum Fallback:** If individual line-item matching fails (e.g., bad OCR on one invoice number reading $10 as $70), the rule calculates the sum of *all found* open invoices referenced in the RMI. If this sum matches the `payment_amount` (within tolerance), it ignores the bad line data and applies the payment.
        4.  **Global Rescue & Client Auto-Correct:** If no open items are found for the linked client, it searches globally for the RMI invoices. If they align perfectly under a *different* single client, it automatically corrects the transaction's client link and updates the `EmailClientLink` learning cache to prevent future fallout.
        5.  **Stale Cache Safeguard:** If no open invoices are found (and global rescue fails) for an email-linked client, it triggers a *live* CMC call (`get_client_by_email`). If the live client ID is *different* from the cached one, it updates the process's `cmd_client_id` and re-runs the stack. Implements a strict `_has_retried` loop guard to prevent infinite ping-pong loops between Global Rescue and Live CMC re-validation.
    * **Conclusive Outputs:** `applied_rmi_direct`, `hitl_needed` (mismatch).

4.  **`ApplyRmiGroupRule`**
    * **Purpose:** Fallback for `ApplyRmiDirectRule` that searches across the entire Client Group.
    * **Logic:** Finds the client's `group_name` (from cache or `OpenInvoice` table). It then performs the same validation as `ApplyRmiDirectRule` (including Total Sum Fallback) but queries `open_invoices` by `client_group == group_name`.
    * **Conclusive Outputs:** `applied_rmi_group`, `hitl_needed` (if the same invoice number is found under multiple clients in that group).

5.  **`ApplyRmiAmountMatchRule`**
    * **Purpose:** RMI Fallback for Garbage IDs. Handles scenarios where RMI contains internal/vendor invoice numbers, but the payment amounts are correct.
    * **Logic:** 1. **Phase 1 (Local):** Greedily matches RMI amounts to Open Invoice Balances OR Original Amounts local to the linked client/group. **Includes CLOSED invoices** to handle timing/sync issues. Uses **LIFO (Newest First)** sorting to prioritize recent debts over older ones with identical amounts.
        2. **Phase 2 (Global):** If amounts remain unmatched, it searches GLOBALLY for those specific amounts. Supports partial application.
    * **Conclusive Outputs:** `applied_rmi_amount_match`, `hitl_rmi_multi_customer_partial`.

6.  **`ApplyBsiSingleFxHintRule`**
    * **Purpose:** Handles cross-currency payments (e.g., DKK paying a USD invoice) where BSI extracted exactly ONE unambiguous invoice hint and the bank omitted the exchange rate from the raw text.
    * **Logic:** 
        1. Calculates the mathematically implied FX rate (Payment Received / Invoice Balance) based on the single extracted invoice.
        2. Performs a live market sanity check via `fx_rate_service.py` (fetching from the Frankfurter API and utilizing Redis caching).
        3. Applies the payment if the implied rate is within a safe 5% tolerance of the daily market rate. (If the API is down, it falls back to a loose bounding box to prevent extreme OCR hallucinations).
    * **Conclusive Outputs:** `applied_bsi_single_fx_hint`.

7.  **`ApplyNettedBsiRule`**
    * **Purpose:** To apply a payment that matches the sum of a *group* of invoices/memos (>= 2) that were extracted directly by BSI.
    * **Logic:**
        1.  Requires `BsiTransaction.extracted_invoice_numbers` to have 2 or more items. Deduplicates identical rows based on BSI array frequency.
        2.  Identifies if it's an FX payment (BSI `instructed_amount` is valid and currency differs from `payment_currency`). Sets `target_amount` and `target_currency` accordingly.
        3.  Finds all matching open items in the client's *group* and `target_currency`. **Includes Closed (0 balance) invoices** in the search so the calculated net sum is mathematically correct.
        4.  Calculates `sum(balance)` and `sum(original_amount)` of the found items.
        5.  **Full Match:** If *all* BSI invoices were found, it checks if `sum(balance)` OR `sum(original_amount)` matches the `target_amount` (within 0.01).
        6.  **Partial Match Tolerance:** If *not all* BSI invoices were found (e.g., some were missing), it checks if the sum of the *remaining* items matches the `target_amount` (within a looser 0.10 tolerance).
    * **Conclusive Outputs:** `applied_bsi_netted`, `applied_bsi_netted_orig_amt`, `applied_bsi_partial_net`, `applied_bsi_netted_fx`.

8.  **`ApplyBsiRawFxRule`**
    * **Purpose:** To apply an FX payment by parsing raw, unstructured text when `ApplyInstructedAmtRule` fails.
    * **Logic:**
        1.  Parses `raw_bank_text` for regex patterns: `SETT-AMT: [CUR] [AMT]` and `RATE: [RATE]`.
        2.  Requires BSI-extracted invoices (`extracted_invoice_numbers`) to be present.
        3.  **Math Check:** Validates that `(Settlement Amount * Rate)` matches the `payment_amount` (within 0.02 tolerance).
        4.  **Application Check:** Validates that the `Settlement Amount` matches the `sum(balance)` or `sum(original_amount)` of the BSI-extracted invoices (which must be in the `Settlement Currency`).
    * **Conclusive Outputs:** `applied_bsi_raw_fx`, `hitl_needed`, `error`.

9.  **`ApplyInstructedAmtRule`**
    * **Purpose:** To apply a 1:1 FX payment using BSI's structured `instructed_amount` field.
    * **Logic:**
        1.  Checks if `BsiTransaction.instructed_amount` and `instructed_currency` are present and different from the payment currency.
        2.  Queries `open_invoices` for the client where `original_amount` == `instructed_amount` and `currency` == `instructed_currency`.
        3.  **BSI Hint Priority:** Strongly prioritizes invoices explicitly extracted by BSI over generic oldest-first matching if multiple items share the same amount.
    * **Conclusive Outputs:** `applied_bsi_instructed_amount`.

10. **`ApplyNetFeeRule`**
    * **Purpose:** To apply a 1:1 payment where a bank fee was deducted or included.
    * **Logic:**
        1.  Finds a *separate* debit (`D`) transaction in the `bsi_transactions` table that shares the *same `bank_reference`* and contains a fee keyword (dynamically fetched from `skip_rules`).
        2.  **Bidirectional Check:** Calculates both `Payment + Fee` (Deducted from principal) AND `Payment - Fee` (Included/OUR instruction).
        3.  It then queries `open_invoices` for the client to find a single item where `balance` matches either calculated target.
    * **Conclusive Outputs:** `applied_amount_client_net_fee`, `hitl_needed` (ambiguous).

11. **`ApplyImplicitFeeRule`**
    * **Purpose:** Resolves cross-border 'lifted' fees (e.g., SWIFT intermediary charges) where the fee isn't explicitly itemized by the bank.
    * **Logic:**
        1. Auto-detects known cross-border transaction codes (e.g., Deutsche Bank '202' or 'ZI28') based on the `IMPLICIT_FEE_CONFIG`.
        2. Tests if adding an expected hidden fee (e.g., 25 EUR) bridges the gap to a perfect invoice balance.
        3. If an exact net match is found, it piggybacks on the existing Net Fee staging logic to stage the missing cash as a distinct fee offset.
    * **Conclusive Outputs:** `applied_amount_client_net_fee`.

12. **`SafeguardBsiInvoicesRule`**
    * **Purpose:** Data Rescue & Safety Gate. Ensures BSI-extracted invoices are not bypassed.
    * **Logic:** 1. Performs a **Live ERP Rescue** via CMC to fetch any unapplied BSI invoices (even if closed in ERP).
        2. **Merges** the newly found (Closed) invoices with existing (Open) matches.
        3. **Client Auto-Correction:** If a rescued invoice belongs to a different client than currently linked, it automatically corrects the client link to match the invoice owner.
        4. Validates the Net Sum of the COMPLETE set (Open + Closed). If it matches, applies immediately (`applied_bsi_netted`).
        5. **Strict Block:** If the sum mismatches or items are truly missing, it strictly blocks the engine from falling back to heuristic (amount-based) matching and forces a HITL review.
    * **Conclusive Outputs:** `applied_bsi_netted`, `applied_closed_invoice`, `hitl_bsi_amount_mismatch`, `hitl_needed`.

13. **`ApplyBsiGroupRule`**
    * **Purpose:** To apply a payment that matches the total sum of *all* open items for a client on a specific timeframe.
    * **Logic:**
        1.  **Anchor:** If BSI extracted an invoice, it finds that invoice's `posting_date` and uses it as the `target_date`.
        2.  **1:1 Yield Guardrail:** If the BSI anchor invoice perfectly matches the payment amount on its own, it strictly yields to standard 1:1 matching rules, bypassing the group sum logic to prevent misclassification.
        3.  **Remainder Logic:** If the extracted invoice is only a partial match, searches for *other* open invoices on that same date to fill the remaining balance. If the exact day fails, it expands to check a **30-day window** (`+/- 30 days` from the anchor).
        4.  **No Anchor:** If BSI extracted no invoices, this rule iterates through *every* open posting date for the linked client to find a `sum` match.
        5.  Checks if `sum(balance)` OR `sum(original_amount)` matches the `payment_amount` (within 0.10 tolerance).
        6.  If multiple groups match (Ambiguous), it explicitly evaluates all matching groups and applies to the **oldest** one.
    * **Conclusive Outputs:** `applied_bsi_group_orig_amt`, `applied_bsi_group_bal`. If no group sum matches *and* no anchor was used, it checks the *total* client balance and can set `hitl_client_total_mismatch`.

14. **`ApplyBsiGroupWithCreditMemoRule`**
    * **Purpose:** Matches payment against the NET SUM of all open items (Invoices + Credit Memos) for a Client Group.
    * **Logic:** Strictly requires that at least one negative balance (Credit Memo) is present in the group. If not, it yields to standard rules to prevent misclassification of simple 1:1 matches.
    * **Conclusive Outputs:** `applied_bsi_group_with_cm`.

15. **`ApplyOrigAmtRule`**
    * **Purpose:** 1:1 match on Original Amount.
    * **Logic:**
        1.  Finds all open invoices for the linked client where `original_amount` == `payment_amount` and `currency` matches.
        2.  **Group-Level Fallback:** If the payment is strictly linked to one entity but no match is found, it gracefully expands its search to the entire *Client Group*. If a unique match is found, it auto-corrects the target client.
        3.  **Crucial Check:** This rule *only* applies if the matched invoice's current `balance` *also* matches the `payment_amount`.
        4.  If `original_amount` matches but `balance` differs (e.g., a partial payment was already made), the rule is *inconclusive* and fails, preventing a misapplication.
        5.  **BSI Hint Priority:** Strongly prioritizes invoices explicitly extracted by BSI over generic oldest-first matching if multiple items match.
    * **Conclusive Outputs:** `applied_amount_client`, `applied_amount_client_oldest`.

16. **`ApplyBalanceMatchRule`**
    * **Purpose:** 1:1 match on Current Balance.
    * **Logic:**
        1.  Finds all open invoices for the linked client where `balance` == `payment_amount` and `currency` matches.
        2.  **Group-Level Fallback:** If no match is found for the specific client, it expands the search to the entire *Client Group*. If a unique match is found, it auto-corrects the target client.
        3.  If one or more matches are found, it applies the payment to the oldest one (by `due_date`/`posting_date`).
    * **Conclusive Outputs:** `applied_balance_client`, `applied_balance_client_oldest`.

17. **`ApplyPaidAmountMatchRule`**
    * **Purpose:** Applies payment that matches the *difference* between Original Amount and Balance.
    * **Logic:** Queries open invoices where `(OriginalAmount - Balance) == PaymentAmount`. This handles cases where the customer pays the "paid portion" or reconciles a specific ledger state.
    * **Conclusive Outputs:** `applied_paid_amt_match`.

18. **`ApplyGlobalOrigAmtRule`**
    * **Purpose:** Global 1:1 Fallback (Client not linked).
    * **Logic:**
        1.  Queries globally for `original_amount == payment_amount`.
        2.  **Baseline Fuzzy Safeguard:** Checks if the bank payer name and the candidate client name are completely dissimilar (e.g., <50% match score). If so, it safely aborts to prevent applying to unrelated clients.
        3.  **1 Match:** Links the client from that invoice and applies the payment.
        4.  **>1 Match (Same Client):** Applies to the oldest.
        5.  **>1 Match (Diff Clients):** Uses an **LLM** (via `vLLMClient` and `rule6_disambiguator` prompt) to compare the BSI Payer Name against the candidate client names. If the LLM returns a single, confident match, it links that client and applies the payment.
    * **Conclusive Outputs:** `applied_amount_global`, `hitl_needed` (ambiguous, LLM failed).

19. **`ApplyPartialBsiRefRule`**
    * **Purpose:** Global Group Fallback.
    * **Logic:**
        1.  Uses `InvoiceParserHelper` to find *only* numeric strings (e.g., '12345') in the BSI raw text that match the *length* of known invoice rules (e.g., a 5-digit rule).
        2.  Reconstructs potential `doc_no` using the rule's prefix (e.g., `G12345`).
        3.  Queries for these reconstructed `doc_no`s *globally*.
        4.  If all found items belong to a **single, unambiguous client group** and their `sum(balance)` matches the `payment_amount`, it links the client and applies the payment.
    * **Conclusive Outputs:** `applied_bsi_ref`, `hitl_needed` (ambiguous).

20. **`OverrideClosedStatusRule`**
    * **Purpose:** Final Gatekeeper. 
    * **Logic:** Runs immediately after a successful application. It verifies if the applied invoices are locally marked as 'CLOSED' or have 0 balance. If ALL applied invoices are closed, it overrides the process status to `applied_closed_invoice`. This ensures the match is recorded but prevents erroneous double-posting to the ERP.

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
    * **Logic:** Checks the `rmi_open_invoice_match` table. If the RMI match *failed* (e.g., `Amount Mismatch`, `Invoice(s) Not Found`), it deduplicates the cached failures, sets the process status to `hitl_rmi_mismatch`, and copies the *failed* invoice list (enriching it with balance/original amount from `open_invoices`) into the `applied_invoices` field for UI review.
    * **Conclusive Output:** `hitl_rmi_mismatch`.

3.  **`DiagnoseBsiFailureRule`**
    * **Purpose:** To provide specific HITL reason for BSI-based group failures.
    * **Logic:** Checks the process `notes` for failure text logged by `ApplyBsiDailyGroupRule` or `ApplyNettedBsiRule` (e.g., "did not match payment").
    * **Conclusive Outputs:** `hitl_bsi_daily_group_mismatch`, `hitl_bsi_amount_mismatch`, `hitl_partial_bsi_match`.

4.  **`DiagnoseClientTotalMismatchRule`**
    * **Purpose:** If no other rule succeeds, checks if the payment exactly matches the client's Total Open Balance (sum of all items). If not, sets a specific mismatch status.
    * **Logic:** Queries the total sum of `balance` and `original_amount` for the linked client. If neither matches the payment amount within tolerance, flags for review.
    * **Conclusive Output:** `hitl_client_total_mismatch`.

5.  **`DiagnoseClientNotFoundRule`**
    * **Purpose:** To provide specific HITL reason if no client was ever linked.
    * **Logic:** Checks if `cmd_client_id` and `matched_client_name` are both `None`.
    * **Conclusive Output:** `hitl_client_not_found`.

* **(Fallback)** If no diagnostic rule applies, the `RuleEngine`'s fallback logic sets the status to `hitl_needed`.
---

## 5. Supporting Services

The Cash Application Assistant utilizes a suite of robust background services and scheduled jobs to handle data ingestion, master data synchronization, proactive matching, and ERP staging. These services operate asynchronously to decouple high-latency tasks (like ERP API calls) from the main orchestration thread.

### 5.1 Smart BSI Processing Poller (`bsi_processing_service.py`)
* **Purpose:** Background service that continuously polls the database for new bank statements and orchestrates the end-to-end cash application pipeline.
* **Concurrency Protection:** Uses Redis distributed locking (`caa_bsi_processing_lock_poller`) with a 60-second timeout to ensure only one worker processes a batch at a time.
* **Adaptive Scheduling:** Reads `settings.yaml` configuration to switch between High-Frequency mode (e.g., 60-second intervals) during specified Peak UTC hours and Low-Frequency mode (e.g., 1800-second intervals) during off-peak hours, optimizing resource utilization.
* **Execution Flow:**
    1. **Open Invoice Pre-Sync:** Triggers a synchronous `run_open_invoice_refresh()` to ensure the local open items cache reflects recent ERP payments before processing new cash.
    2. **RMI Match Recovery (`_refresh_recent_rmi_matches`):** Re-evaluates recent (last 7 days) RMI records that previously failed matching (e.g., `Amount Mismatch`, `Not Found`). This resolves timing discrepancies where remittance emails arrive hours or days before the corresponding invoices are generated or opened in the ERP.
    3. **Ingestion & Filtering:** Fetches a batch of `unprocessed` transactions from `bsi_transactions` (up to `batch_size_limit`). Automatically flags transactions belonging to system-ignored bank accounts (e.g., `hidden_accounts`) as `ignored`.
    4. **CMC Safety Gate (`_validate_and_prune_batch`):** * Extracts all raw invoice numbers parsed from the current batch of bank texts.
        * Performs a bulk live validation against the ERP (`cmc_connector.validate_invoices`).
        * **Circuit Breaker:** If the CMC connection fails or times out, the entire batch processing cycle is aborted to prevent data loss or false HITL routing.
        * **Pruning:** Successfully validated numbers are kept; invalid "noise" numbers (e.g., phone numbers matching the regex length) are permanently pruned from the `BsiTransaction.extracted_invoice_numbers` array, and the transaction is marked as `VALIDATED`.
    5. **Process Initiation:** Iterates through the validated batch, creating a new `CashApplicationProcess` state machine for each transaction and immediately invoking `process_next_step`.

### 5.2 Scheduled Open Invoice Sync (`caa_main.py` / `open_invoices.py`)
* **Purpose:** Ensures the local `open_invoices` cache remains perfectly synchronized with the Oracle ledger's state.
* **Trigger:** Managed via `APScheduler` using `CronTrigger` aligned to specific UTC hours (1, 5, 9, 13, 17, 21) at the 45th minute. This schedule is designed to execute 15 minutes *after* standard Oracle daily sync jobs complete.
* **Concurrency Protection:** Implements distributed **Redis Locking** (`caa_scheduled_sync_lock`) with a 10-minute timeout. This prevents concurrent execution across multiple server workers (e.g., Gunicorn processes), avoiding DDOSing the CMC connector which could result in partial data loads and falsely mark thousands of active invoices as "Closed".

### 5.3 Linking Service (`linking_service.py`)
* **Purpose:** Proactively links unmapped RMI sender emails to CMD clients to populate the `email_client_link` cache, drastically speeding up the main orchestrator's matching phase.
* **Trigger:** Runs continuously in a background thread (default 300-second interval) protected by a Redis lock.
* **Process Flow & Enhancements:**
    1. **Aggregator Interception:** Checks the database `aggregators` table. If the email belongs to a known third-party payor (e.g., Bill.com, Cass Info), it bypasses entity linking entirely. It creates an "Aggregator Mode" link (`is_aggregator = True`, `cmd_client_id = None`) to force downstream multi-customer invoice splitting.
    2. **Internal Mail Routing:** Hardcoded exclusion for internal automated emails (e.g., `billing@gtt.net`, `no-reply@gtt.net`), linking them to a dummy "GTT Internal" client to prevent infinite matching loops.
    3. **Live ERP Lookup:** Calls `cmc_connector.get_client_by_email`. Handles HTTP 429 (Rate Limit) and gRPC Deadline Exceeded errors by triggering an automatic 60-second cooldown sleep.
    4. **Fuzzy Resolution Logic:** * Evaluates CMC candidates against the RMI-provided `customer_name` using `fuzz.token_set_ratio` (Threshold: 85).
        * If CMC returns an `EXACT_MATCH`, but the fuzzy score suggests a different candidate in the payload is a better semantic match, the system utilizes tie-breaker logic to override the exact domain match.
        * If CMC returns a `DOMAIN_MATCH` with multiple active clients, the system relies on the fuzzy score. If scores are ambiguously tied, it falls back to the client with the highest `contact_count` in the ERP.
    5. **Persistent Storage:** Records the resolution (Success, Ambiguous, Timeout, Rate Limit) into `email_client_link` with the specific `match_method_used` string for auditability.

### 5.4 Oracle Staging Service (`oracle_staging_service.py` & `oracle_rules/`)
* **Purpose:** Translates logical cash applications into strict Oracle accounting entries. Manages the `oracle_receipt_queue` database table (the Reconciliation Workbench).
* **Architecture:** Utilizes a **1:N Split Staging** model. A Strategy Pattern evaluates the transaction context and executes specific rules (`StandardStagingRule`, `BofaStagingRule`, `NetFeeStagingRule`, `FxStagingRule`, `VarianceStagingRule`) to explode a single bank payment into multiple individual receipt lines (one per invoice).
* **Core Capabilities:**
    * **Sequence Generation:** Generates continuous Oracle Receipt Numbers (e.g., `DB9141.02212026.001`) using the account's last 4 digits. Explicitly prevents excluded transactions (Fees, Internal) from consuming sequence numbers, maintaining a gap-free audit trail.
    * **Balance Reconciliation:** Extracts Opening/Closing balances directly from the BSI file headers and calculates real-time variance against the sum of the staged queue (`Posted + Ready + HITL + Excluded`).
    * **Lockbox Detection (`BofaStagingRule`):** Detects `CHECK` and `LOCKBOX` op-codes. Separates summary deposits (Code 115) with a `SUM` prefix to prevent double-posting, and individual checks with a `CHK` prefix (or exact check number parsed from Addenda). Forces the receipt method to `GTT Lockbox Receipts`.
    * **Cross-Currency FX Logic (`FxStagingRule`):** When payment currency differs from invoice currency, calculates the exact FX rate (`bsi_ledger_amount` / `bsi_original_amount`). Stages the Oracle receipt in the Ledger Currency for bank reconciliation, while writing the application reference in the Foreign Currency. Overridden cross-currency receipts explicitly scale down the ledger amount to match the clamped invoice amount, ensuring the ERP calculates the correct market FX rate and preventing false exchange losses.
    * **Cross-BU Bridging & Asynchronous Queuing:** Identifies if an applied invoice belongs to a Business Unit distinct from the receiving bank account's Oracle BU. Automatically flags the receipt type as `cross_bu`. To handle Oracle PROD environments where Intercompany (IC) receipts are generated by scheduled background processes, the orchestrator intercepts the `Pending_Oracle_Automation` status, stages it as `Pending IC Bridge`, caches routing variables (`cross_bu_context` JSONB) into a Redis Sorted Set (ZSET), and waits for the asynchronous micro-worker to finish application. If delayed beyond 60 minutes, it routes to `HITL (IC Bridge Timeout)`. A follow-up REST POST is executed after successful SOAP IC application to forcefully attach the Invoice Reference metadata to the automated Oracle Child Receipt.
    * **Credit Memo Segregation & Safety:** Isolates negative target amounts or explicit 'CR' prefixes into a distinct `credit_memo` receipt type (Status: `cm_pending`). Ensures they do not consume the main payment waterfall cash total. A strict safety gate drops self-referencing CM applications (e.g., applying CR123 to target invoice CR123) to prevent opaque Oracle errors.
    * **Credit Memo BAN Priority:** CM waterfall netting logic strictly prioritizes netting against positive invoices that share the *exact same Customer BAN* before evaluating remaining items. Exact allocation amounts are recorded explicitly inside the CM's `references` array during initial staging, replacing sibling guesswork during the final Oracle push.
    * **Underpayment, Ghost Receipts, & Bank Fees:** Uses a strict waterfall logic prioritizing explicitly selected invoices. If funds run out, unpaid invoices generate "Ghost Receipts" (Status: `underpayment`, Prefix: `DEFICIT_`). If a positive cash remainder perfectly matches a BSI `charges_amount`, it auto-stages the remainder as a 'Bank Fee Offset'. Implicit bank fees and short payments apply a consistent `DEFICIT_FEE_` prefix to non-cash fee deductions, ensuring these rows are excluded from ledger cash sums while maintaining proper ERP offsets. Explicit bank fees (`charges_amount`) are prioritized over internal calculations to stage cross-currency fees perfectly.
    * **SWIFT/ISO Charge Bearer Logic:** Fee generation deterministically relies on the SWIFT/ISO `charges_bearer` field (`OUR`, `SHA`, `BEN`) from the BSI to automatically decide whether to stage additive cash-consuming `FEE_` receipts or non-cash `DEFICIT_FEE_` offsets.
    * **Historical Lookups:** For transactions matched to already-paid invoices (`STATUS_APPLIED_CLOSED_INVOICE`), the service queries the live ERP to retrieve the *actual* historical Oracle Receipt Number that originally closed the invoice, embedding this in the UI.
* **Bi-Directional Sync Protocol & Safety Gates:**
    * Executes a **SOAP Atomic** request (`CreateStandardReceipt`) to post fully applied receipts directly to invoices. The orchestrator explicitly drops `AmountReceiptCurrency` and `ExchangeRate` payload fields when passing a 1.0 normalized rate for same-currency transactions to prevent Oracle API rejections.
    * Immediately issues a **REST verification** fetch (`GetStandardReceipts`) to retrieve the generated `StandardReceiptId`, `CreationDate`, and confirmation `State`, syncing the local staging record.
    * **Global Prefix Blocks:** The Oracle push engine enforces strict security blocks that automatically intercept and reject any attempt to post receipts containing the `DEFICIT_` or `CLOSED_` prefixes, protecting the ERP from data corruption. Dummy staging records for already paid invoices explicitly use the `CLOSED_` prefix to allow safe manual split/re-allocation.
    * **Smart Batch Posting Safety Gate:** During bulk posting, the system intelligently evaluates variance at the *individual transaction level* rather than locking the entire account batch. Mismatched transactions are dynamically disabled and flagged in the Post Batch dialog.
    * **Smart Unapplied Routing:** Intelligently detects if a receipt has zero invoice targets (e.g., Overpayments) and automatically routes the API push to the 'Unapplied / On Account' ERP workflow. The backend API correctly inherits the exact BAN from sibling invoices within the same bank transaction.

### 5.5 RMI Invoice Matching Service (`invoice_matching_service.py` & `rmi_rule_engine.py`)
* **Purpose:** Proactively evaluates unlinked Remittance Advice records against the open invoice ledger. Pre-calculates match viability to populate the `rmi_open_invoice_match` cache, accelerating the main orchestrator and providing immediate feedback in the Remittance UI.
* **Trigger:** Background loop running on an interval, protected by a Redis lock (`caa_invoice_matching_lock`).
* **Retry & Backoff Architecture:**
    * Implements smart exponential backoff based on the previous failure reason.
    * Errors or system timeouts (`Error`, `Pending`) retry up to 3 times, with a 1-hour interval between attempts.
    * Logical failures (`Not Found`, `Amount Mismatch`) retry only 1 time, with a 24-hour interval, allowing time for batch invoice generation in the ERP to catch up to early remittance emails.
* **Pre-Processing & Deduplication:** Before executing rules, it scans the RMI invoice list. If identical invoice numbers are present (e.g., due to line-item discounts), it deduplicates them by keeping only the entry with the largest absolute amount, filtering out $0.00 noise.
* **The `RmiRuleEngine` Execution Chain:**
    The engine executes the following discrete rules in strict priority order. If a rule returns a conclusive status, the engine stops.

    1. **`MatchClientInvoicesRule` (Primary Local Match)**
        * Validates that the parsed RMI invoice numbers exist locally for the email-linked client.
        * **Currency Validation:** Fails the invoice if the open item currency differs from the RMI payment currency.
        * **Closed Invoice Check:** If the invoice is found but has a 0 balance or is marked `CLOSED`, it flags it as `Matched (Closed)`.
        * **Amount Matching:** Checks if the RMI paid amount perfectly matches the `balance` (within 0.01 tolerance) -> `Matched (Balance)`. If not, checks the `original_amount` -> `Matched (Orig Amt)`.
        * **Global Yield Safeguard (Amount Fallback Hijacking):** The local amount fallback matching expands its exact match search scope to the entire `client_group` and implements a 'Global Yield Safeguard' to verify global uniqueness before falling back to heuristics.
        * **Ambiguity Handling (Oldest First):** If the RMI amount matches *multiple* local invoices by original amount (and the balances are not exact matches), it sorts the candidates by `due_date` then `posting_date` (ascending) and applies the payment to the oldest open debt.
        * Fails the item (`Amount Mismatch`) if the invoice is found but neither balance nor original amount matches.

    2. **`MatchLiveInvoiceRule` (Live ERP Fallback)**
        * If `MatchClientInvoicesRule` determines that specific invoice numbers are completely missing from the local database, this rule kicks in.
        * Calls `cmc_connector.validate_invoices` to check the live ERP.
        * If found in the ERP (even if they are fully paid/closed or zero-balance), it creates new `OpenInvoice` records in the local database and immediately performs the amount/balance matching logic described above.

    3. **`MatchGlobalInvoiceRule` (Aggregator / Mislinked Fallback)**
        * Runs if local matching failed. Discards the linked client context and searches for the exact RMI invoice numbers *globally* across the entire database.
        * **Unique Match:** If exactly 1 global invoice is found per RMI number, and the amounts match, it succeeds (`Matched (Balance)` or `Matched (Orig Amt)`).
        * **Ambiguous Match:** If a single RMI invoice number belongs to multiple different clients globally (e.g., invoice "1001"), it flags the match as `Amount Mismatch (Ambiguous: Inv# Found for Multiple Clients)`.

    4. **`MatchClientTotalBalanceRule` (No RMI Invoices)**
        * Executed if the RMI parsed successfully but contained zero valid invoice numbers.
        * Checks if the total payment amount on the remittance perfectly matches the `balance` of a single, unique open invoice for the linked client.

    5. **`MatchClientGroupedBalanceRule` (Daily Sum Fallback)**
        * Executed if the total balance rule fails.
        * Aggregates all open invoice balances for the linked client, grouped by `posting_date`. 
        * If the total RMI payment exactly matches the sum of a specific day's invoice run, it succeeds (`Part of Daily Group`).

    6. **`MatchGlobalOriginalAmountRule` (Final Gross Fallback)**
        * Searches globally for any single open invoice where the `original_amount` exactly matches the gross RMI payment amount.
        * **Normalized ID Tie-Breaker:** If multiple global invoices share the exact same original amount, the rule extracts all non-numeric characters and leading zeros from the RMI reference strings (e.g., converting "INV-00123" to "123") and compares them against normalized versions of the candidate `doc_no` strings to find a unique, fuzzy ID match.

* **Email Link Learning:** If the matching engine achieves a highly confident status (`Matched`, `Matched (Balance)`, `Matched (Orig Amt)`) and a client ID was successfully determined from the invoice ownership, it triggers `_learn_email_link_from_match`. This automatically maps the sender's email to that Client ID in the `EmailClientLink` table, effectively teaching the main orchestrator the correct routing for future payments.

### 5.6 Smart Match Service (`smart_match_service.py`)
* **Purpose:** Provides intelligent invoice application suggestions specifically for Human-in-the-Loop (HITL) items within the Oracle Staging UI.
* **Logic:** When a user initiates a manual match, the service pulls the remaining transaction amount, the original instructed gross amount (if different), and any "Hints" (invoice numbers extracted explicitly by the BSI parser). It runs an algorithm across all open items for the candidate BAN/Client:
    1. **BSI Hint Check:** Evaluates permutations of BSI-extracted hints.
    2. **Combinatorial Math:** Checks all possible combinations of up to 4 open invoices.
    3. **Date Block Analysis:** Checks contiguous sequential blocks of invoices (e.g., invoices from the same week).
    4. **Strict FIFO:** Falls back to oldest-first fulfillment if no mathematical combination perfectly sums to the target amount.

### 5.7 Gateway Service & Workload Analytics (`gateway_service.py`)
* **Purpose:** Integrates with the centralized SERA API gateway to fetch the active directory list of authorized Finance operators (e.g., `SERA-Group-Collections`), utilizing a 5-minute in-memory cache to prevent network saturation.
* **Volume Analytics:** Evaluates the trailing 90-day history of bank transactions (`BsiStatementFile` joined with `BsiTransaction`) to analyze operational volume. It automatically categorizes accounts into health tiers (Active, Auto-Excluded, Dormant, Hidden) based on actionable cash received, allowing system administrators to map "Daily Default Assignees" strictly to active, revenue-generating ledgers.

### 5.8 Event-Driven Polling Engine (`cross_bu_worker.py`)
* **Purpose:** A highly efficient, lightweight micro-worker specifically handling Oracle Intercompany (IC) Bridge automation delays.
* **Mechanism:** Utilizes Redis Sorted Sets (ZSETs) to manage delayed tasks. It polls for Oracle IC child receipt generation every 2 minutes with zero idle database load. If the child receipt isn't generated within **60 minutes**, it safely times out and routes the item to the `HITL (IC Bridge Timeout)` queue for manual intervention.

### 5.9 Market FX Rate Service (`fx_rate_service.py`)
* **Purpose:** Provides daily market exchange rates to validate and sanity-check OCR and parsing logic on cross-currency transactions.
* **Mechanism:** Fetches daily rates from the Frankfurter API and utilizes Redis caching to prevent API rate-limit spam, providing a baseline ground truth for rules like `ApplyBsiSingleFxHintRule`.

## 6. Manual Intervention & Learning System

The system is designed to improve over time via Human-in-the-Loop (HITL) interactions. The UI provides specialized diagnostic workflows to resolve ambiguous cases and seamlessly resume automation.

1.  **Manual Client Linking & Persistent Payer Learning:**
    * When the orchestrator fails to identify a client (`hitl_client_not_found`), users can manually link a raw Payer Name to a CMD Client ID via the UI dialog (`ManualClientLinkDialog`).
    * **Persistence:** This creates a record in the `payer_client_links` table.
    * **Automation:** The `MatchClientByManualLinkRule` runs *first* in the linking phase, ensuring the system "remembers" this correction for all future payments from that exact payer name.

2.  **Manual Invoice Application & Smart Match (AI):**
    * For transactions flagged with amount mismatches or missing invoices, users can utilize the "Find Invoices" dialog to manually search for and apply open (or closed) invoices.
    * **Smart Match Engine:** Integrated directly into the UI, users can trigger an AI-driven suggestion engine. It uses combinatorial math, date-block analysis, and BSI-extracted hints to automatically select the optimal combination of invoices that perfectly sum to the payment amount.
    * **Waterfall Split Trigger:** Clicking "Apply" on selected invoices instantly triggers a waterfall split logic, generating individual receipts for the selected invoices and a residual "On Account" line for any remaining balance.

3.  **Manual RMI Resolution (Ambiguity Handling):**
    * When the system encounters a `hitl_multiple_rmi_candidates` status (e.g., identical payment amounts on the same day for different entities like Amazon Sweden vs. Amazon Poland), the `ResolveMultipleRmiDialog` is presented.
    * Operators can inspect conflicting Remittance Advice candidates side-by-side (viewing Sender Email, Date, and Invoice Counts) and manually select the correct one to resume the automated matching sequence.

4.  **Categorization & Exclusion:**
    * Users can mark transactions as "Tax", "Fees", or "Internal Transfer" directly in the Oracle Staging or Cash Application views.
    * **Rule Creation:** The UI allows converting these one-off actions into permanent `SkipRule` database entries. These rules support multi-criteria AND logic (e.g., matching Payer Name AND Transaction Text simultaneously) and can be scoped globally or to specific Bank Groups.

5.  **Auto-Feedback Loop & Global Analysis:**
    * When a user manually edits a receipt amount or invoice application in the Staging area, `feedback_processor.py` captures the "diff". This diff explicitly tracks and displays the exact invoice numbers added or removed during manual UI splits to provide precise context.
    * It uses an LLM (`vLLMClient` powered by Gemma-4-31B-it) to analyze *why* the user made the change (e.g., "User added $15 fee to match the invoice balance") and logs it for system tuning.
    * **Raw Context Viewer:** The auto-feedback engine explicitly captures and persists the exact JSON state snapshot (`context_data`) before and after a user modification. Operators can view this raw data payload directly in the Feedback Inbox to better understand the AI's analysis.
    * **Enhanced Visibility:** The auto-generated feedback summary explicitly states the starting status of the receipt (e.g., `Receipt ACH2019... (in 'hitl')`) and captures the `display_name` of the specific user who modified the receipt, prepending their name to the diff summary.
    * **Cash Application Process Deep Linking:** Users can create feedback tickets directly from bank transactions on the Cash Application Process page. The ticket includes a bank icon that navigates to the exact Cash Application Process item, featuring a hover description: "View Linked Cash Application Process: {cash_appliction_process_id}".
    * **Screenshot Integration:** Operators are able to paste screenshots when submitting a feedback ticket in the FeedbackDialog. Images are stored in `ai/cash_application_assistant/assets/faq/` and displayed in the submitted ticket with click-to-zoom capability. Deleting the ticket from the UI permanently deletes the database record and the associated image file.

6.  **Dedicated Transaction Audit Logging:**
    * High-impact structural system events (Manual Splits, Deletes, Live ERP Syncs, Restages, Auto-Bridges) and all operator field modifications are permanently recorded in the isolated `transaction_audit_logs` table.
    * This event-sourced audit trail keeps the Feedback Inbox strictly focused on AI analysis and UX feedback, while natively powering the Operator Leaderboard Analytics.

---

## 7. Oracle Staging & Reconciliation Workbench

The **Oracle Staging Viewer** acts as a final firewall before data enters the ERP. It provides a robust, month-based windowing interface (lazy-rendered for performance) for Finance users to review, correct, and commit cash applications.

### Key Features

1.  **Split Staging Architecture (1:N):**
    * Migrated from a 1:1 model to a 1:N model. A single bank transaction is exploded into a separate Oracle receipt for every individual invoice matched, improving ERP reconciliation accuracy.

2.  **Balance Reconciliation Logic & Transaction-Level Variance Flagging:**
    * The view pulls **Opening Balance** and **Closing Balance** directly from the BSI Statement File (`bsi_statement_files` table).
    * It calculates a **Projected Closing Balance** by summing the amounts of all cash transactions in the staging queue (`Posted`, `Ready`, `HITL`, `Excluded`). Credit Memos are ignored in this sum as they are non-cash adjustments.
    * **Variance Check:** It compares the `Projected` vs `Actual` Closing Balance.
        * **Green:** Zero Variance (The queue perfectly matches the bank statement).
        * **Red:** Variance detected (Funds are missing from the queue or amounts are incorrect).
    * **Transaction-Level Variance Flagging:** The transaction header calculates the sum of all staged cash items in real-time (explicitly excluding all `DEFICIT_` and `fees` prefix rows) and flags mismatches against the original bank ledger amount with a warning icon and high-visibility red styling. This allows operators to instantly isolate problematic transactions (e.g., ghost duplicates or manual entry errors) without relying solely on the account-level balance summary.

3.  **Sequence Logic & Gap Prevention:**
    * Oracle Receipt Numbers are generated automatically using the pattern `{BankPrefix}{Last4}.{MMDDYYYY}.{Sequence}` (e.g., `DB9141.11272025.001`).
    * **Exclusion Logic:** Transactions marked as `Excluded`, `Fees`, or `Internal` do **not** consume a sequence number. This ensures that the final posted receipts in Oracle have a continuous, gap-free sequence (e.g., `.001`, `.002`, `.003`).

4.  **Smart Status Grouping & Visual Identifiers:**
    * **Active Items:** Receipts that require action (`Ready`, `HITL`, `Error`) are grouped at the top.
    * **Credit Memos:** Identified and isolated with distinct Pink styling (`cm_pending`). They do not consume the main payment waterfall cash total.
    * **Lockbox Transactions:** Detected via BSI addenda (Code 115 summaries vs individual checks) and marked with distinct Purple styling. They are strictly routed to the 'GTT Lockbox Receipts' method.
    * **Ghost Receipts & Bank Fees:** If funds run out during waterfall application, unpaid invoices generate "Ghost" receipts (Prefix: `DEFICIT_`). Non-cash fee deductions utilize the `DEFICIT_FEE_` prefix. Both ghost receipts and explicit `fees` rows are universally excluded from ledger cash variance math while maintaining proper accounting offsets for the ERP.
    * **Cross-Transaction Duplicate Highlighting:** The UI automatically detects and visually flags invoices that are targeted multiple times within the same daily batch. Duplicate invoice numbers are highlighted in red with a warning icon to alert operators to overlapping payments before ERP posting.

5.  **Granular Task Assignment & Audit Trail:**
    * **Batch & Item Assignment:** Users can "Grab" an entire batch (all receipts for an account/date) or individual transactions.
    * **Smart Assignment Routing & Load Balancing:** Workload assignments can be split within a single bank account based on the Oracle Receipt Prefix (e.g., 'ACH', 'WIRE'). When multiple operators share a rule, the system load-balances receipts using an MD5 hash of the Customer BAN to preserve customer continuity. A calendar-aware 'Monday Catch-up' logic automatically pushes assignments for the trailing 3 weekend days.
    * **Locking & Overrides:** Assigned items display a "Locked" chip with the owner's name. Users can release their own locks or perform a "Force Override" on others.
    * **My Work View:** A "My Assignments Only" toggle instantly filters the workbench to strictly isolate and display only the user's active workload blocks.
    * **Auditability & Deletion Logs:** Every row persists `assigned_to_user_id`, `last_updated_by`, and `posted_by` fields. Additionally, deleting a receipt requires a mandatory reason that is permanently recorded, along with the operator's name, in the Feedback Inbox.

6.  **Advanced Edit Capabilities & Excel-Mode Splitting:**
    * Users can modify **Receipt Number**, **Comments**, and **Currency** via a modal dialog.
    * **Smart Edit Auto-Split:** If an operator manually reduces the cash amount on a receipt (e.g., to pay a specific invoice), the system automatically generates a new 'Overpayment' receipt for the remainder to enforce a strict 1:1 match with the bank deposit total. The 'Applied Invoices' field is strictly read-only to prevent bypassing the staging logic; the dialog is also fully CM-aware, unpacking and repackaging internal target offsets without data loss.
    * **Split & Allocate Dialog:** A spreadsheet-like interface allowing users to split a single bank transaction into unlimited receipt lines across multiple BANs. It features an "Auto-Balance" calculator and explicitly differentiates between 'Cash' receipts and 'Non-Cash' offsets (like Credit Memos and 'Bank Fee Offset'). A soft warning allows operators to safely override mathematically unbalanced cash allocations.
    * **Bulk Import:** Users can copy and paste transaction splits directly from Excel or CSV into the dialog with auto-delimiter detection and an 'Overwrite existing rows' toggle.
    * **Placeholder Safeguard:** If an operator deletes the last remaining receipt row for a transaction, the system instantly spawns a full-amount 'Placeholder' receipt in `HITL` status, preventing the transaction from vanishing and causing confusing ledger variances.

7.  **Credit Memo Management & Waterfall Application:**
    * Clicking a Credit Memo opens the `CreditMemoDetailsDialog` to view live application history and optionally **Unapply** specific invoices directly from the UI. The UI actively prevents operators from accidentally saving a Credit Memo that targets its own receipt number.
    * During batch posting, Credit Memos utilize a **Waterfall Application Strategy**: They distribute their value across multiple sibling receipts (prioritizing identical BANs first, then highest amounts) if the credit balance exceeds a single target invoice.

8.  **Cross-BU Bridging & Cross-Currency Display:**
    * **Cross-BU:** Detects when a payment targets an invoice in a different Business Unit. Stages as a `Cross-BU` transaction (`cross_bu_pending`), displaying a `Pending IC Bridge` UI state with a spinning Sync icon while waiting on Oracle's scheduled automation. If delayed beyond 60 minutes, it falls to `HITL (IC Bridge Timeout)`. Upon posting, it displays both the **Parent Receipt** and **Child Receipt** IDs.
    * **Cross-Currency:** Displays both the Ledger Currency (e.g., RON) and the Foreign Invoice Currency (e.g., EUR) simultaneously, performing real-time FX math to balance the ledger. Manually overriding a cross-currency receipt proportionally scales down the ledger amount to match the clamped invoice amount, ensuring the ERP calculates the correct market FX rate and averting false exchange losses.

9.  **Live Oracle Validation & Historical Context:**
    * **Posted Receipts:** Clicking a receipt number fetches the live JSON payload from Oracle, confirming accounting dates and states.
    * **Closed Invoices:** For transactions matched to already-closed invoices, the UI dynamically fetches and displays the *actual historical Oracle Receipt Number* that originally closed the invoice. Clicking it reveals exactly who/what paid the invoice previously.
    * **High-Severity Visibility:** Transactions matched against already-paid invoices are heavily styled with a bold red error background and 'ALREADY PAID' status chip to prevent accidental double-posting.

10. **Advanced Search & Interactive Filtering:**
    * A powerful global search bar allows finding transactions by Receipt Number, Payer Name, BAN, IBAN, Amount, or Invoice Number. Features 1-Click Copy icon buttons next to Receipt and Invoice numbers.
    * **Extended Quick Filters & Instant Daily Search:** Operators can filter the staging view by `Bank Fees`, `Underpayments (Deficits)`, `Overpayments`, and `Transaction Type` (Lockbox, Credit Memos, Standard). An Instant Daily Quick Filter search bar isolates receipt number, amount, BAN, or client name across the currently loaded batch with zero server lag.
    * **Transaction-Aware Filters:** Clicking summary pills filters the view but intelligently preserves the **full transaction context**. Sibling receipts remain visible alongside the error.
    * **Sorting:** Transactions within an account group can be sorted dynamically by Amount (Low to High / High to Low), Customer, BAN, or Receipt Number.

11. **Bi-Directional Sync & Direct Posting:**
    * **Push & Safety Gate:** Bulk-posts "Ready" and "CM_Pending" items to Oracle via the ORC Connector using **SOAP Atomic** requests. The Smart Batch Posting Safety Gate evaluates variance at the individual transaction level (dynamically disabling and flagging mismatched transactions in the dialog) rather than locking down the entire account batch.
    * **Smart Unapplied Routing:** Intelligently detects if a receipt has zero invoice targets (e.g., Overpayments) and automatically routes the API push to the 'Unapplied / On Account' ERP workflow, resolving the BAN dynamically.
    * **Live ERP Rescue on Regeneration:** The 'Regenerate Receipts' action on blocked transactions performs an instant live check against the Oracle ERP, fetching restored balances immediately (bypassing the 6-hour sync delay) and staging the receipt as 'Ready'.
    * **Sync & Idempotency Trap Fix:** Users can manually sync specific receipts. The Oracle Read-After-Write verification tracks the newly minted `StandardReceiptId` directly from the SOAP creation payload to the REST verification fetch, guaranteeing exact 1:1 retrieval and avoiding old, reversed receipts.

12. **Transaction History Timeline:**
    * A comprehensive, chronological timeline view showing the entire lifecycle of a transaction from initial AI orchestration, through manual edits and splits, to final ERP posting.
    * **Time Travel State Reconstruction:** Parses historical audit logs to accurately display the exact original state (Amount, BAN, Invoices, Status) of a receipt the moment it was initially staged, prior to any manual user modifications.
    * **Audit Payload Visibility:** Captures and logs the exact JSON payload sent to the ERP even when an API call fails, dramatically improving debugging for Oracle rejection errors (e.g., missing invoices, closed periods).

13. **Oracle Remittance Name Headers:**
    * The staging account group headers display the full Oracle Remittance Name (e.g., 'Deutsche Bank 68 EUR 9000') alongside the BSI Account Number as a muted chip. This allows operators to instantly identify and reconcile bank accounts using familiar ERP naming conventions.

---

## 8. User Interface (React SPA)

The CAA UI is built with React, Vite, and Material UI, featuring a robust Service-Oriented Architecture (SOA) and Feature-Based Module Structure. Code is cleanly separated into domain features (`cash-application/`, `oracle/`, `rules/`, `bank-statements/`, `remittances/`).

### Key Modules & Views

* **Monitoring Dashboards:**
    * **Operational Dashboard:** High-level executive statistics, workflow performance, and invoice match rule usage.
    * **Staging History Widget:** Provides a rolling 30-day breakdown of Daily Performance Trends (Success vs. HITL vs. Excluded receipts), offering clear visibility into auto-application rates.
    * **Export Dashboard:** Generates a comprehensive multi-sheet Excel report containing Executive Summaries, Staging Volumes, BSI Headers (balances), and raw BSI Transaction logs for offline analysis.
* **Remittances (RMI) & Bank Statements (BSI):**
    * Views for ingested emails and bank transactions.
    * **Ingestion Schedule Widget:** A predictive timeline in the BSI view that analyzes 7-day trailing history to forecast estimated arrival times for upcoming bank files (e.g., "10:00 UTC - 3 Banks").
    * **Manual Upload:** Support for operators to manually upload Bank of America US Excel statements directly via the UI, bypassing scheduled batch runs.
* **Intervention Tools:**
    * Actions for bulk reprocessing (filtered by date/bank/status), manual transaction splitting, triggering live RMI re-matches, and forcing email-to-client re-links.
* **Business & Processing Rules Configuration (`/rules`):**
    * **Business Units & Accounts:** A hierarchical view to manage Oracle configuration, linked physical bank accounts, and CMD invoice parsing regex rules.
    * **Skip Rules:** Manage complex, multi-criteria exclusion logic.
    * **Client Links:** Manage and audit persistent manual Payer-to-Client overrides.
    * **Aggregators:** Dynamic management of third-party payment aggregator definitions (e.g., Bill.com, Cass Info) and their expected routing prefixes, eliminating the need for code deployments to add new payment processors.
    * **Workload Assignments:** Interface for the Collections team to set daily default assignees for specific bank accounts. Powered by underlying **Volume Analytics**, which categorizes accounts into health tiers (Active, Auto-Excluded, Dormant, Hidden) based on 90-day transaction history, ensuring resources are allocated only to actionable ledgers.

---

## 9. Future Enhancements & TODO Plan

### 9.1 Functional Roadmap
* **Direct Posting Mode:** Implement a "Fast Track" configuration that allows high-confidence matches (e.g., 100% Exact Match on Invoice & Amount) to bypass the Oracle Staging Queue and post directly to ERP, further reducing manual review volume.
* **Archive Function:** Once Oracle is fully processed we should "archive" all related items from our database
* **Unapplied Receipt Recovery Flow:** Create a new dedicated UI flow for the specific scenario where a receipt header is successfully created in Oracle (leaving it unapplied) but the invoice application fails. Implementation details:
  * **Dedicated Status:** Add a new staging status `hitl_unapplied` to `OracleReceiptStatusChip.jsx` with a distinct warning style to differentiate it from standard unapplied receipts.
  * **"Apply to Invoices" or "Mark as OK" Action:** Conditionally render action buttons in `OracleReceiptActions.jsx` when the status is `hitl_unapplied`, allowing operators to either retry invoice application or mark the receipt as OK (leaving it unapplied intentionally) without recreating the header.
  * **Backend Wiring:** Update the ORC Connector to return `partial_success=True` and the `StandardReceiptId` when application fails. Expose a new CAA endpoint (e.g., `POST /oracle/apply-invoices`) that bypasses header creation and explicitly calls the `ApplyStandardReceipt` gRPC wrapper.
* **Regenerate ID Preservation:** Once the Unapplied Receipt Recovery flow is built and the Oracle connector is made idempotent, update the 'Regenerate' and 'Reprocess' logic to safely preserve and reuse existing Oracle receipt IDs rather than generating new ones. (Currently kept as "new ID generation" to intentionally avoid deadlock crashes).

### 9.2 Documentation & AI Context Optimization
* **Glossary of Concepts:** Create a strict definition list distinguishing *BSI Payer*, *CMD Client*, and *Oracle Customer* to prevent LLM hallucinations during feedback analysis.
* **Failure Mode Documentation:** Explicitly document the *inverse* conditions for each rule (e.g., "Why did `ApplyNettedBsiRule` fail?") to help the AI explain the "Why" to users.
* **Ambiguity Logic Maps:** Document the specific tie-breakers used in client linking (Recency vs. Context vs. LLM) so the AI can explain why a specific client was chosen over another.
* **Status Translation Layer:** Create a mapping table that translates internal technical statuses (e.g., `hitl_bsi_amount_mismatch`) into user-friendly explanations for the AI to use in chat responses.
* **Configuration Traceability:** Explicitly map system behaviors to their controlling database tables (`skip_rules`, `invoice_rules`) to help the AI suggest configuration fixes.