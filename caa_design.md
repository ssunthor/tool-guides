# Cash Application Assistant (CAA) - Design Document

**Version:** 1.0
**Date:** 2025-10-29
**Author(s):** Martin Kecka, Gemini AI
**Category:** Assistant (A)
**Acronym:** CAA

---

## 1. Introduction & Purpose

The **Cash Application Assistant (CAA)** is an AI-enhanced orchestration module designed to automate the process of matching incoming bank statement transactions (payments) to corresponding remittance advices and open invoices. Its primary goal is to determine how a received payment should be applied, reducing manual effort and improving the speed and accuracy of cash application.

CAA operates by processing individual bank transactions ingested by the Bank Statement Ingestor (BSI). For each transaction, it executes a sequence of predefined rules to:

* Identify and Skip Non-Payment Transactions: Filter out items like bank fees, payroll, or internal transfers.
* Link to Supporting Data: Find related Remittance Advice details processed by the Remittance Mail Ingestor (RMI).
* Identify the Client: Determine the correct client entity associated with the payment using data from RMI, BSI details, and the Client Master Connector (CMC).
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
* **CMC (Client Master Connector):** Provides client lookup (via email) and open invoice data via gRPC calls (GetClientByEmail, GetOpenInvoices, GetClientAllData).
* **PostgreSQL Database:** Stores CAA's internal state (cash_application_process table), linking information (email_client_link), rule definitions (business_units, bank_accounts, invoice_rules), and cached results (rmi_open_invoice_match). Also reads data from BSI and RMI tables.

### Outputs / Interactions:

* **PostgreSQL Database:** Updates the cash_application_process table with status changes, linked data (client ID, RMI ID), applied invoices, and processing notes. Writes results of background linking to email_client_link and invoice matching to rmi_open_invoice_match.
* **Redis:** Used for distributed locking (linking_service) to prevent concurrent runs of background tasks.
* **LLM Service (AI Factory):** Used by specific rules (e.g., Rule 6 disambiguation) and the feedback analysis feature.
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

### Orchestration (process_next_step)

* For a given `process_id`, this core function retrieves the current `CashApplicationProcess` record.
* It checks if the current status is terminal (e.g., 'applied\_...', 'skipped', 'hitl\_needed', 'error'). If so, it stops.
* Based on the current status, it sequentially executes a predefined set of Orchestration Rules (detailed in Section 4).
* Each rule attempts to gather more information (like finding an RMI, linking a client, or identifying invoices) and potentially update the process's status, notes, and linked data (e.g., `cmd_client_id`, `applied_invoices`).
* If a rule reaches a conclusive outcome (successfully applies the payment, determines it should be skipped, identifies an error, or decides HITL is needed), the orchestrator commits the changes to the database and stops for this `process_id`.
* If a rule is inconclusive (e.g., couldn't find an RMI, couldn't link a client yet), the orchestrator proceeds to the next rule in the sequence.
* If all applicable rules run without a conclusive outcome, the status is set to `hitl_needed`.

### Background Tasks

* **Periodic Linking (linking\_service.py):** Runs periodically to find new sender emails in RMI results that haven't been linked to clients yet. It calls CMC and updates the `email_client_link` table. This provides cached data used by Rule 2.
* **Periodic RMI Invoice Matching (invoice\_matching\_service.py):** Runs periodically to attempt matching RMI results directly against open invoices, storing the outcome in `rmi_open_invoice_match`. This data is primarily used by the Cash App process rules (e.g., Rule 4, 4.5) but also provides insights on the Remittance page UI.

---

## 4. The Orchestration Rules (Detailed Logic)

The core logic resides in the sequential execution of rules within `process_next_step`. Each rule checks preconditions (current status, available data) before executing.

### Rule 0: Skip Condition Check (skip\_rules.py)

* **Purpose:** Identify and filter out transactions that are clearly not standard customer payments early in the process.
* **Logic:** Checks the associated `BsiTransaction` details (payer name, descriptions, amount, transaction type) against predefined keyword lists and conditions.
* **Input State:** `new`
* **Output States:**
    * `fees`: Keyword match for bank fees/charges. (Conclusive)
    * `payroll`: Keyword match for payroll/salary. (Conclusive)
    * `internal_transfer`: Match for known internal company names or specific BSI transaction types. (Conclusive)
    * `ap_transfer`: Transaction amount is negative (debit). (Conclusive)
    * `skipped`: Match for general skip keywords (e.g., TAX, RENT) or missing payer name. (Conclusive)
    * `error`: If the associated `BsiTransaction` cannot be found. (Conclusive)
* **Conclusive:** Yes, if any skip condition is met. If not met, allows progression.

### Rule 1: Find Matching RMI Record (client\_linking\_rules.py)

* **Purpose:** Find a relevant Remittance Advice (RMI) record that likely corresponds to this bank transaction, primarily to get the sender email for client linking.
* **Logic:** Queries `RmiProcessingResult` for records matching the payment amount (within tolerance) and currency, within a date window (-7/+2 days) around the `payment_date`. Prefers the closest match by date that has a sender email.
* **Input State:** `new` (after Rule 0 passes)
* **Output States:**
    * Sets `rmi_processing_id` on the `CashApplicationProcess` if a match is found.
    * Sets status to `pending_client_email` if RMI is found.
    * Sets status to `pending_client_invoice_link` if no RMI is found.
* **Conclusive:** No. Determines the next step (email linking vs. BSI invoice linking).

### Rule 2: Match Client via RMI Email (client\_linking\_rules.py)

* **Purpose:** Identify the CMD client using the sender email from the linked RMI record.
* **Logic:**
    * Checks the `email_client_link` table for a cached link for the normalized RMI sender email.
    * If a valid cached link exists, uses it and sets status to `client_linked`.
    * If a cached failed lookup exists and is recent (within 1 day), sets status to `hitl_needed`.
    * If no cache hit or cached failure is old, calls `cmc_connector.get_client_by_email`.
    * Processes the CMC response (handles exact match, domain match, fuzzy comparison logic as implemented in `linking_service.process_single_email`).
    * Updates the `email_client_link` cache table with the result (success or failure).
    * If a unique client is identified, updates `cmd_client_id`, `matched_client_name`, and sets status to `client_linked`.
    * If CMC confirms no match, sets status to `hitl_needed`.
    * If CMC call fails (error, timeout, rate limit), sets status to `error` (or `hitl_needed` if specifically a rate limit/timeout that might resolve).
* **Input State:** `pending_client_email`
* **Output States:**
    * `client_linked`: Client found via cache or live CMC call. (Proceeds to application rules)
    * `hitl_needed`: No client found (confirmed by cache or live call), or ambiguous match. (Conclusive)
    * `error`: CMC connector error or other unexpected issue. (Conclusive)
    * `pending_client_invoice_link`: If email linking fails inconclusively (e.g., temporary CMC error allowing fallback).
* **Conclusive:** Yes, if status becomes `client_linked`, `hitl_needed`, or `error`. Otherwise, allows fallback.

### Rule 3: Match Client via BSI Invoice Reference (client\_linking\_rules.py)

* **Purpose:** Identify the client by finding invoice numbers mentioned in the BSI transaction details (reference, raw text) and looking them up in the `open_invoices` table.
* **Logic:**
    * Retrieves `reference_details` and `raw_bank_text` from the associated `BsiTransaction`.
    * Uses `InvoiceParserHelper` to extract potential invoice/credit memo numbers based on rules defined for the receiving bank account's Business Unit.
    * Queries `open_invoices` for the extracted document numbers.
    * If all found open invoices belong to a single client group, links the process to the primary client name found within that group.
* **Input State:** `pending_client_invoice_link` (if Rule 1 found no RMI, or Rule 2 failed inconclusively)
* **Output States:**
    * `client_linked`: Unique client group identified. (Proceeds to application rules)
    * `hitl_needed`: Found invoices map to multiple client groups (ambiguous). (Conclusive)
    * `pending_client_fuzzy_link`: No invoices extracted, or extracted invoices not found/matched uniquely. (Inconclusive)
* **Conclusive:** Yes, if `client_linked` or `hitl_needed`.

### Rule 3.5: Match Client via Fuzzy Payer Name (client\_linking\_rules.py)

* **Purpose:** As a fallback, attempt to link the client by fuzzy matching the `payer_name_raw` from the BSI transaction against distinct `client_name` values in the `open_invoices` table.
* **Logic:**
    * Retrieves `payer_name_raw`.
    * Queries distinct `client_name` from `open_invoices`.
    * Uses `fuzzywuzzy.token_set_ratio` to compare the payer name against each distinct client name.
    * If a single client name scores above a threshold (e.g., 85) and is unambiguous, links the process to that client name.
* **Input State:** `pending_client_fuzzy_link` (if Rule 3 failed)
* **Output States:**
    * `client_linked`: Unique fuzzy match found. (Proceeds to application rules)
    * `client_linked` (Implicitly): If ambiguous fuzzy match, proceeds to application rules without setting client info yet (application rules might identify client).
    * `client_linked` (Implicitly): If no fuzzy match, proceeds to application rules without setting client info.
* **Conclusive:** No. Always proceeds to application rules, potentially setting `client_linked` status.

### Rule 4: Apply RMI Invoices (Direct Client) (invoice\_application\_rules.py)

* **Purpose:** If a client is linked and an RMI was found, attempt to apply the payment based on the invoices listed in the RMI matching exactly (number and amount) against open items for that specific client.
* **Logic:**
    * Extracts invoice numbers and amounts paid from the linked `RmiProcessingResult`.
    * Queries `open_invoices` for these doc numbers, filtered by the linked `client_name` and `payment_currency`.
    * Validates if all RMI invoices exist as open items for the client and if the `amount_paid` matches the `balance` (within tolerance).
    * Checks if the sum of RMI `amount_paid` matches the BSI `payment_amount` (within tolerance).
    * Includes a re-validation step: If invoices aren't found for an email-linked client, it re-queries CMC live to check if the link is stale, potentially updating the client and retrying Rule 4.
* **Input State:** `client_linked`
* **Output States:**
    * `applied_rmi_direct`: All conditions met. (Conclusive)
    * `hitl_needed`: Mismatches found (missing invoices, amount differences, payment discrepancy). (Conclusive)
* **Conclusive:** Yes.

### Rule 4.5: Apply RMI Invoices (Client Group) (invoice\_application\_rules.py)

* **Purpose:** If Rule 4 failed (likely because invoices weren't found under the directly linked client), check if the RMI invoices match open items belonging to any client within the linked client's group.
* **Logic:**
    * Determines the `client_group` of the linked client.
    * Queries `open_invoices` globally for the RMI invoice numbers, filtered by `client_group` and `payment_currency`.
    * Checks for ambiguity (same invoice number existing under multiple clients within the group).
    * If unambiguous and all RMI invoices are found under a single (potentially different) client within the group, and amounts/totals match, apply the payment.
* **Input State:** `client_linked` (after Rule 4 is inconclusive)
* **Output States:**
    * `applied_rmi_group`: All conditions met. Updates `matched_client_name` if different from originally linked. (Conclusive)
    * `hitl_needed`: Ambiguous matches within the group. (Conclusive)
* **Conclusive:** Yes, if applied or ambiguous. Otherwise, inconclusive.

### Rule 4.8: Apply Netted BSI Invoices (invoice\_application\_rules.py)

* **Purpose:** Apply payment if multiple invoice/memo numbers extracted directly by BSI (in `extracted_invoice_numbers`) exactly net out to the payment amount within a single client group.
* **Logic:**
    * Checks if `BsiTransaction.extracted_invoice_numbers` contains 2 or more entries.
    * Queries `open_invoices` for these document numbers, filtering by `payment_currency` and potentially by the linked `client_group` (if known).
    * Verifies all found open items belong to the same client group.
    * Calculates the sum of `balance` for these items.
    * If the sum matches `payment_amount` (within tolerance), applies the payment.
* **Input State:** `client_linked` (after Rule 4.5 is inconclusive)
* **Output States:**
    * `applied_bsi_netted`: Match found. Links client if not already linked based on matched items. (Conclusive)
    * `hitl_needed`: Ambiguous (invoices span multiple groups) or amount mismatch. (Conclusive)
* **Conclusive:** Yes, if applied, ambiguous, or error. Otherwise, inconclusive.

### Rule 5: Apply Single Exact Original Amount (Client - Oldest First) (invoice\_application\_rules.py)

* **Purpose:** Apply payment if the `payment_amount` exactly matches the `original_amount` of a single open invoice for the linked client. Prioritizes the oldest invoice if multiple matches exist.
* **Logic:**
    * Queries `open_invoices` for the linked `client_name` and `payment_currency`, where `original_amount` matches `payment_amount`.
    * Orders results by `due_date`, then `posting_date`.
    * If one or more matches found, applies payment to the first (oldest) result.
* **Input State:** `client_linked` (after previous application rules are inconclusive)
* **Output States:**
    * `applied_amount_client`: Exactly one match found. (Conclusive)
    * `applied_amount_client_oldest`: Multiple matches found, applied to oldest. (Conclusive)
* **Conclusive:** Yes, if applied. Otherwise, inconclusive.

### Rule 5.5: Apply Amount Minus Fee (Client) (invoice\_application\_rules.py)

* **Purpose:** Apply payment if the `payment_amount` plus the amount of a single, related fee transaction (identified by matching `bank_transaction_id` and fee keywords) exactly matches the `balance` of a single open invoice for the linked client.
* **Logic:**
    * Finds potential fee transactions (debit, < 0) linked by `bank_transaction_id` in `bsi_transactions`.
    * If exactly one fee transaction is found, calculates `net_amount = payment_amount + fee_amount`.
    * Queries `open_invoices` for the linked `client_name` and `payment_currency`, where `balance` matches `net_amount`.
    * If exactly one match found, applies the `net_amount`.
* **Input State:** `client_linked` (after Rule 5 is inconclusive)
* **Output States:**
    * `applied_amount_client_net_fee`: Unique match found. (Conclusive)
    * `hitl_needed`: Multiple open items match the net amount (ambiguous). (Conclusive)
* **Conclusive:** Yes, if applied or ambiguous. Otherwise, inconclusive.

### Rule 5.8: Apply Based on Instructed Amount (Client) (invoice\_application\_rules.py)

* **Purpose:** Apply payment if the instructed amount and instructed currency from the BSI transaction exactly match the `original_amount` of a single open invoice for the linked client. This handles payments made in a different currency than the invoice.
* **Logic:**
    * Retrieves `instructed_amount` and `instructed_currency` from the `BsiTransaction`.
    * Queries `open_invoices` for the linked `client_name`, matching `instructed_currency` and where `original_amount` matches `instructed_amount`.
    * Orders by age (due/posting date).
    * If one or more matches found, applies the actual `payment_amount` received to the oldest matching invoice.
* **Input State:** `client_linked` (after Rule 5.5 is inconclusive)
* **Output States:**
    * `applied_bsi_instructed`: One or more matches found (applied to oldest). (Conclusive)
* **Conclusive:** Yes, if applied. Otherwise, inconclusive.

### Rule 6: Apply Single Exact Original Amount (Global) (invoice\_application\_rules.py)

* **Purpose:** If no client-specific rules applied, attempt to find a single, unique open invoice globally where the `original_amount` and currency match the payment, regardless of client linkage. Uses LLM for disambiguation if multiple candidates exist.
* **Logic:**
    * Queries `open_invoices` globally where `original_amount` and currency match the payment.
    * If exactly one match, links the client from that invoice and applies the payment.
    * If multiple matches for different clients:
        * Uses an LLM (`OllamaClient` with `rule6_disambiguator` prompt) to compare the `payer_name_raw` against the candidate client names.
        * If LLM returns a single, confident match, links that client and applies payment to their oldest matching invoice.
        * If LLM fails or is uncertain ("None"), sets status to `hitl_needed`.
    * If multiple matches for the same client, applies to the oldest one.
* **Input State:** Any state where application hasn't succeeded yet (e.g., `client_linked`, `pending_client_fuzzy_link` after failed fuzzy match).
* **Output States:**
    * `applied_amount_global`: Unique global match found (or disambiguated by LLM/same client). Links client if necessary. (Conclusive)
    * `hitl_needed`: Ambiguous match across different clients, and LLM failed or Payer Name was missing. (Conclusive)
* **Conclusive:** Yes, if applied or HITL. Otherwise, inconclusive.

### Rule 7: Apply Partial BSI Invoice Reference (invoice\_application\_rules.py)

* **Purpose:** Attempt to reconstruct potential full invoice numbers from partial numbers found in BSI text (using BU-specific rules) and apply payment if the sum of balances for uniquely identified open items matches the payment amount.
* **Logic:**
    * Uses `InvoiceParserHelper` to find numeric strings in BSI text that match the length requirements of rules for the account's BU.
    * Reconstructs potential full `doc_no` using rule prefixes.
    * Queries `open_invoices` globally for these potential `doc_no` values and the `payment_currency`.
    * Filters results based on the linked `client_name`, if available.
    * If all found items belong to a single client (or client group if not linked yet) and their balance sum matches the `payment_amount`, applies the payment. Links client if identified via this rule.
* **Input State:** Any state where application hasn't succeeded yet.
* **Output States:**
    * `applied_bsi_ref`: Match found. Links client if necessary. (Conclusive)
    * `hitl_needed`: Ambiguous (matches multiple clients/groups) or amount mismatch. (Conclusive)
* **Conclusive:** Yes, if applied or HITL. Otherwise, inconclusive.

### Fallback

* **Purpose:** If none of the above rules result in a conclusive success, skip, or error state.
* **Logic:** Sets the process status to `hitl_needed`.
* **Input State:** Any state that hasn't reached a conclusive outcome after all applicable rules run.
* **Output State:** `hitl_needed` (Conclusive)
* **Conclusive:** Yes.

---

## 5. Supporting Services

### Linking Service (linking\_service.py)

* Runs periodically in the background (controlled by `caa_main.py`).
* Uses Redis for distributed locking to ensure only one instance runs at a time.
* Queries `rmi_processing_results` for distinct sender emails not present in `email_client_link`.
* For each new email, calls `cmc_connector.get_client_by_email`.
* Applies fuzzy matching logic (using RMI customer name vs. CMC results) to select the best client match or determine failure/ambiguity.
* Inserts/Updates the result (success or failure) into the `email_client_link` table. This table acts as a cache for Rule 2.
* Includes delays and handles rate limiting/timeouts from CMC.

### RMI Invoice Matching Service (invoice\_matching\_service.py)

* Runs periodically in the background (controlled by `caa_main.py`).
* Queries `rmi_processing_results` for records eligible for matching (e.g., successful RMI status) that either haven't been matched before, are 'Pending', or have an old 'Error'/'Not Found' status eligible for retry.
* For each RMI record, it executes a subset of the main orchestrator's matching logic (`_perform_match` function, primarily Rules 1-5).
* Upserts the result (status, matched invoices, method) into the `rmi_open_invoice_match` table. This provides data for the UI and potentially for Rule 4/4.5 if needed, although the main orchestrator often runs its own checks.

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
* **Enhanced LLM Use:** Explore using LLMs for more complex disambiguation scenarios or potentially for suggesting applications in HITL cases based on historical data.
* **UI Improvements:** Add more detailed reporting, potentially user-assignable tasks for HITL items, and richer visualizations.