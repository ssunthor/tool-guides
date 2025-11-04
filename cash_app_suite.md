# High-Level Architectural Design: Cash Application Suite (SERA)

**Version:** 1.0.0
**Date:** 2025-11-04
**Author(s):** Gemini AI, Martin Kecka
**Goal:** Full automation of cash application, from ingestion to Oracle posting.

---

## 1. Overview & Architectural Pattern 🧠

The project successfully implements the automation of the cash application process based on the **Delegated Autonomy** SERA pattern. The entire business process lifecycle is delegated to the **Cash Application Assistant (CAA)**, which acts as the intelligent orchestration layer (the "AI Accountant").

* **SERA Pattern:** **Delegated Autonomy**.
* **Central Intelligence:** **Cash Application Assistant (CAA)**.
* **Primary Trigger:** Event-driven, initiated by the **Bank Statement Ingestor (BSI)** upon successfully loading a new transaction into the shared database.
* **Shared Data Layer:** All modules read and write to a shared **PostgreSQL database**, which houses the core BSI/RMI transaction data and the critical **DB-Driven Business Rules** (`bank_accounts`, `invoice_rules`).

---

## 2. SERA Modules and Core Functionality 🛠️

The architecture consists of four implemented modules (RMI, BSI, CMC, CAA) and the planned Oracle Connector (ORC).

### 2.1. Remittance Ingestor (RMI) - (I) Ingestor

* **Purpose:** Monitors the `msgraph` inbox in real-time to extract and interpret unstructured remittance advice (PDFs, Excel, images).
* **Ingestion Pipeline:** Executes a robust, sequential pipeline:
    * **Text Extraction Gauntlet (Step 2):** A failover sequence attempting the best extraction method first (Excel $\rightarrow$ Paged-PDF $\rightarrow$ OCR $\rightarrow$ Marker $\rightarrow$ Nanonets).
    * **Stage A4 Validation & Correction Loop:** The AI-powered self-healing stage. It triggers multi-pass vision models to repair both incorrect invoice *numbers* and *amounts* if `validate_totals` fails.
* **Enrichment:** Uses **DB-driven helpers** (`BankHelper`, `InvoiceHelper`) to validate and enrich extracted data against the live `bank_accounts` and `invoice_rules` tables, replacing brittle static files.
* **Data Output:** Writes structured remittance data, including **invoice line item JSONB**, to the `rmi_processing_results` table.
* **Key Feature:** The **`refine_sender_email`** step uses an LLM to find the *original external sender* of a forwarded message, crucial for client linking.
* **Future Goal:** Implement the **Profile-Driven Intelligent Pipeline** to use sender-specific, optimized workflows instead of the full gauntlet.

### 2.2. Bank Statement Ingestor (BSI) - (I) Ingestor

* **Purpose:** Automates the ingestion, parsing, and standardization of multi-format bank statement files (CSV, XLSX) from various banks.
* **Channels:** Supports a **gRPC API** for programmatic uploads (saves to temp, archives on success) and a **Local Directory Scanner** for batch processing.
* **DB-Driven Enrichment (Core Value):**
    * `BranchHelper`: Queries the `bank_accounts` table to resolve account numbers to human-readable **branch names**.
    * `InvoiceParserHelper`: Queries the `invoice_rules` table to fetch regex patterns (prefix/length) to extract structured **invoice/credit memo numbers** from raw transaction text.
* **Data Output:** Creates records in `bsi_statement_files` (summary), `bsi_transactions` (lines), and **upserts** balances into `daily_account_balances` (for time-series data).
* **Key Feature:** The `reprocess-statement --file-id <UUID>` CLI command finds the original file in the archive, re-parses it, and updates/deletes related database records to ensure data integrity after parsing logic changes.

### 2.3. Client Master Connector (CMC) - (C) Connector

* **Purpose:** The authoritative, resilient gateway to the **CMD GraphQL API** for client and master data.
* **Dual Service Model:** Serves both:
    * **Live GraphQL API** results: `GetClientByEmail` (handles domain matching and disambiguation) and `ValidateInvoices`.
    * **Local Asset Data:** Pre-processed snapshots (`live_open_invoices.csv`, `invoice_rules.json`) served via gRPC for high-speed access.
* **Resilience:** Built on the `tenacity` library, managing OAuth2 token refresh and implementing specific backoff strategies for **HTTP 429 (Rate Limit)** errors.
* **Future Goals:**
    * Implement **User Identity Auditing** by logging the `X-Forwarded-User` header for all transactions.
    * Develop the **Panda** analysis utility (`GetAggregatedInvoices`) to return small, pre-processed summaries instead of large, raw datasets, preventing LLM overload.

### 2.4. Cash Application Assistant (CAA) - (A) Assistant

* **Purpose:** The **Brain**. Embodies the complex cash application decision-making logic, orchestrating the end-to-end process and managing exceptions.
* **Core Logic:** The **RuleEngine** (`core/rule_engine.py`) operates as a state machine on the `cash_application_process` table, guided by the transaction's current status.
* **Pre-Processing Services:** Runs dedicated background services (`linking_service`, `invoice_matching_service`) to proactively populate lookup caches, improving real-time performance.
* **Application Rules Stack (Priority-Driven):** Executes a high-priority sequence of rules in the `client_linked` state:
    * **`ApplyRmiDirectRule`:** Highest confidence match using pre-validated RMI data.
    * **`ApplyOrigAmtRule` / `ApplyBalanceMatchRule`:** 1:1 invoice matches (original or current balance).
    * **`ApplyNettedBsiRule`:** Matches the payment amount to a sum of invoices/memos extracted by BSI.
    * **`ApplyGlobalOrigAmtRule`:** Global 1:1 match that uses an **LLM** for **contextual client disambiguation** if multiple clients match the amount.

### 2.5. Oracle Connector (ORC) - (C) Connector (Future)

* **Status:** Planned future enhancement (Pending ORC development).
* **Purpose:** To provide a safe, deterministic, and LLM-free gRPC API for all necessary actions within the Oracle ERP system.
* **Key gRPC Methods (Planned):** `CreateReceiptBatch`, `AddPaymentToBatch`, `FindOpenItems`, `ApplyPaymentToItems`, `PostBatch`.
* **Integration Point:** The final execution step of the CAA RuleEngine for all successfully applied transactions (`applied_...` statuses).

---

## 3. End-to-End Automated Workflow (The CAA State Machine) ➡️

The workflow is driven by the state of the `CashApplicationProcess` record, executing a specific set of rules at each stage:

1.  **Trigger (N/A):** **BSI** completes processing and inserts data into `bsi_transactions`, creating a new `cash_application_process` record with status `new`.
2.  **State `new`:**
    * `CheckSkipConditionsRule` filters non-payment items (fees, payroll, internal transfers).
    * `FindRmiRule` attempts to find a matching `RmiProcessingResult` (based on amount, currency, and date window).
3.  **State `pending_client_email`:**
    * `MatchClientByEmailRule` uses the RMI sender email and **CMC** to resolve the `cmd_client_id` (sets status to `client_linked`).
4.  **State `pending_client_invoice_link`:** (Fallback if RMI/Email failed)
    * `MatchClientByBsiInvoiceRule` uses **BSI's enriched invoice numbers** to query CMC open items and find an unambiguous client group (sets status to `client_linked`).
5.  **State `pending_client_fuzzy_link`:** (Fallback if all structured methods failed)
    * `MatchClientByPayerPrefixRule` and `MatchClientByFuzzyPayerRule` attempt to link the client via name analysis.
6.  **State `client_linked`:** (The Main Application Stack)
    * The **Application Rules** (Section 2.4) run in sequence until a conclusive match is found (e.g., `applied_rmi_direct`).
7.  **State $\rightarrow$ `pending_hitl_diagnosis`:** (If no application rule concludes)
    * **Diagnosis Rules** (`DiagnoseRmiFailureRule`, `DiagnoseClientNotFoundRule`) execute to determine the *specific* reason for failure (e.g., `hitl_rmi_mismatch`, `hitl_client_not_found`).
8.  **Final Execution (`applied_...`):** (Future) The process calls **ORC.ApplyPaymentToItems()** and **ORC.PostBatch()** to complete the transaction in the ERP.

---

## 4. Human-in-the-Loop (HITL) Integration 🧑‍💻

HITL is managed as a highly structured exception workflow, with the CAA providing a specific diagnosis before pausing.

* **Trigger:** The CAA enters the `pending_hitl_diagnosis` state after all application rules fail.
* **Diagnosis:** The CAA runs a final set of rules to set a highly specific **diagnostic status** (e.g., `hitl_bsi_amount_mismatch` if `ApplyBsiDailyGroupRule` logged a failure).
* **Escalation:** The workflow is paused, and a structured request is sent (e.g., to a dedicated UI) containing:
    * All gathered context (BSI, RMI, CMC data).
    * The precise, actionable **diagnostic status** code.
* **Resolution:** The CAA waits for a structured response (via API) from the human operator, which provides the final application instructions, allowing the CAA to update the database and resume the workflow.