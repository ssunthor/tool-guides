# Treasury Cash Assistant (TCA) - Design Document (Draft)

**Version:** 0.1.0
**Date:** 2026-06-30
**Author:** Martin Kecka
**Category:** Assistant (A)
**Acronym:** TCA

---

## 1. Introduction & Purpose

The **Treasury Cash Assistant (TCA)** is an AI-enhanced workflow automation module designed to automate the daily treasury cash reporting process. 

Currently, the GTT Treasury department relies on manual, complex Excel formulas to parse Oracle bank statements, calculate USD-equivalent cash positions, and explain daily cash movements. This manual process takes over an hour per day and introduces significant key-person dependencies and risk of error.

The primary goal of TCA is to replace this fragile process with a deterministic, rule-based categorization engine backed by an AI interface. It automatically categorizes all incoming bank statement lines and generates consolidated cash reports ready for Treasury review within minutes. 

**Target Users:** Treasury, GL Cash Team (Accounting), Cash Apps Team.

### 1.1 The "Perfect Answer" Scenario (AI Assistant Job)
TCA provides natural language querying capabilities. When a Treasury user asks for the cash position, TCA is designed to deliver a response such as:
> "The cash position for the GTT group on June 20th was $Xm, with $Xm available once removing restricted cash and outstanding checks. Cash is split between USD $X, EUR $X, GBP $X and Other $X. Cash balances are held between $Xm in the US and $Xm International. 
>
> The cash position between [Date] and [Date] changed by $Xm, caused primarily by $Xm in AP disbursements and $Xm in payroll payments, which was greater than $Xm in customer receipts during the period. As per the forecast, the cash position, AP and payroll were as forecasted.
> 
> For same day cash collections, we have received $Xm in collections as of [time]... The largest customer receipts include XXXX, XXXXX and XXXXX."

---

## 2. Architecture & Integration

* **Service Type:** (A) Assistant Module operating as a dedicated systemd service on a KVM virtual machine.
* **Core Function:** Ingests bank statements, categorizes transaction lines via deterministic rules, computes FX equivalents, generates cash reports, and serves responses via a Human-in-the-Loop (HITL) UI and NLP chat interface.

### 2.1 Data Sources / Inputs (The "Free Lunch" Layer):
TCA does not integrate directly with bank portals; it leverages the broader SERA ecosystem's ingestors:
* **BSI (Bank Statement Ingestor) / ORC:** TCA subscribes to the existing Oracle bank statement feeds. It consumes parsed `bsi_transactions` securely fetched by the ORC and BSI services.
* **FX Feed (Multi-Source Strategy):** TCA utilizes a flexible multi-source strategy for daily exchange rates. The primary goal is to leverage official rates from **Oracle** (via ORC) or **Global Pay (GP)** APIs. However, to mitigate historical access delays with internal teams, the system actively incorporates the public **Frankfurter API** as a robust, immediate alternative (mirroring its successful integration in the Cash Application Assistant).
* **Manual Forecast Ingestor:** A dedicated Python watcher/parser that ingests the Treasury team's Cash Forecast data (e.g., via CSV/Excel uploads).
* **Email Connector (Check Numbers):** An IMAP/Exchange listener designed to extract outstanding check numbers sent via email to adjust available cash calculations.
* **Config Data:** Lists of restricted bank accounts and manual rules configured by Treasury.

### 2.2 Outputs / Interactions:
* **PostgreSQL Database:** The central state engine and historical ledger.
* **vLLMClient (Gemma-4-31B-it):** Used strictly as a Natural Language to SQL/JSON translator for the chat interface. **It does not perform financial math.**
* **User Interface (React SPA):** A dedicated Treasury Workbench for HITL review, categorization overrides, rule management, and natural language querying.

---

## 3. Core Functionality & Process Flow ⚙️

TCA operates as a daily state machine driven by the `status` field in the `tca_cash_process` table (e.g., `new` -> `pending_categorization` -> `pending_review` -> `approved`).

### Phase 1: Ingestion & Normalization
1. **Trigger:** The daily Oracle bank statement sync completes via BSI.
2. **Data Fork:** TCA clones the raw transaction payload for the specified Treasury accounts.
3. **Ancillary Data Sync:** TCA fetches the current day's FX rates, parses the latest Forecast data, and queries the email queue for outstanding checks.
4. **Validation Check:** The system verifies if all expected bank statements have arrived. If a statement is missing, a critical alert is triggered for the Treasury team.

### Phase 2: The Categorization Engine
Similar to the Cash Application Assistant (CAA), this engine evaluates every bank statement line against Treasury's predefined logic.
* **Dynamic Rule Application:** Uses JSONB multi-criteria AND/OR logic (Exact Match, Contains, Regex) against fields like `payer_name`, `transaction_text`, and `bank_reference`.
* **Categorization Output:** Tags each line into specific Treasury buckets:
  * AP Disbursements
  * Payroll
  * Customer Receipts
  * Intercompany
  * Restricted Cash
  * Other
* **Fallback Logic:** If a line fails all rules, it is tagged as `Uncategorized` and the daily report status is immediately flagged as `hitl_categorization_needed`. A default value is used to explicitly flag to Treasury that this line needs amendment.

### Phase 3: Calculation & Aggregation
Once all lines are categorized, TCA computes the daily position:
1. **FX Normalization:** Converts EUR, GBP, and 'Other' currency buckets into USD equivalents using the daily Oracle rate.
2. **Geographic Splitting:** Segregates cash balances into US vs. International based on the physical bank account metadata.
3. **Net Available Cash:** Subtracts `Restricted Cash` and `Outstanding Checks` from the gross position.
4. **Variance Analysis:** Compares the actual categorized outflows/inflows against the parsed Cash Forecast data.

### Phase 4: Human-in-the-Loop (HITL) Review
TCA strictly enforces a review stage before any report is finalized or issued.
* The system halts and presents the drafted report and all categorized lines in the React UI.
* Treasury operators review the activity compared to opening/closing balances.
* Operators manually assign `Uncategorized` lines or override AI decisions.
* Once validated, the user clicks "Approve," locking the daily materialized view and allowing the report to be issued.

---

## 4. The NLP Query Interface (Chat)

To fulfill the use case requirement for natural language interactions (e.g., *"Explain why the cash moved in the way it did"*), TCA implements a strict **RAG-to-SQL pattern**:

1. **User Input:** Treasury user asks a question in the UI.
2. **Intent Parsing:** The LLM translates the query into a structured SQL query directed *only* at the locked, approved `tca_daily_cash_position` materialized views.
3. **Data Retrieval:** The deterministic PostgreSQL database executes the query and returns the exact math.
4. **Natural Language Generation:** The LLM receives the hard data payload and formats it into the required "Perfect Answer" narrative structure.

---

## 5. Supporting Services

TCA utilizes several background services to automate data gathering, decoupling high-latency tasks from the main orchestrator.

* **BSI Sync Poller:** A background task that polls the BSI module for new, unprocessed statement files. It includes a validation gate to ensure all expected daily accounts are present before triggering Phase 1.
* **Scheduled FX Sync (`fx_sync_service.py`):** Runs daily to query exchange rates using a multi-source fallback approach (Oracle, Global Pay API, and Frankfurter API) and persists them to the local database for USD-equivalent calculations.
* **Forecast Watcher (`forecast_ingestor.py`):** Monitors a designated upload directory for manual Cash Forecast CSV/Excel files and safely merges them into the `tca_cash_forecast` table.
* **Email Ingestor (`outstanding_checks.py`):** Periodically scans a designated Treasury mailbox to extract lists of outstanding checks, storing them as offsets against the daily balance.

---

## 6. Data Storage & Schema

TCA maintains its state and historical records in a PostgreSQL database.

* **`tca_cash_process`**: The core state machine tracking the daily progression (`new`, `pending_review`, etc.).
* **`tca_categorized_lines`**: The granular ledger of classified bank transactions, containing the original BSI references, the matched rule ID, and the assigned category.
* **`tca_daily_cash_position`**: Materialized views storing the final, immutable calculated cash balances, FX equivalents, and geographic splits.
* **`tca_rules`**: The dynamic ruleset managed via the UI, storing JSONB logic for categorization.
* **`tca_cash_forecast`**: Parsed manual forecast data used for variance reporting.
* **Data Retention:** All records, including audit logs, are retained for a minimum of 3 years to comply with financial auditing requirements.

---

## 7. Audit, Compliance & Security

* **Strict Access Control:** Standard users (e.g., Cash Apps) are granted **Read-Only** access to view reports and categorization data. 
* **Edit Permissions:** Only designated Treasury Management personnel (e.g., Scott Smith, Kimberly Snader, Tim Fryzer-Smith) have the authority to edit categorization rules, upload forecast data, or override AI decisions.
* **Immutable Logging:** The `tca_audit_logs` table records every structural change. It tracks who created or edited a categorization rule, who uploaded forecast data, when it was updated, and captures the previous/new states.
* **Go-Live Readiness:** Final deployment requires formal sign-off from the Director of Treasury and the Treasurer.

---

## 8. Phased Implementation Strategy

TCA will be delivered in two major phases to mitigate risk and align with data availability.

### Phase 1: Previous-Day Reporting (Target: 3 Months)
* **Scope:** Automate cash reporting using *previous-day* bank statement data from Oracle. 
* **Deliverables:** Establish KVM infrastructure, DB schemas, BSI/ORC ingestion pipelines, Forecast parser, UI Workbench, and the Core Categorization Engine. 
* **Value:** Significant reduction in daily manual Excel processing.

### Phase 2: Same-Day Reporting (Target: End of 2026)
* **Scope:** Incorporate *same-day* reporting capabilities by utilizing intraday bank statements.
* **Dependency:** Since Oracle may not natively provide MT942 intraday statements, Treasury must secure direct banking APIs or automated MT942 feeds (via BSI) to bypass Oracle entirely and fetch real-time transaction data on demand.
* **Deliverables:** Expanded UI for intraday snapshots and parallel state machines for mid-day cash collections reporting.

---

## 9. Risks, Constraints, and Assumptions

* **Risk (Manual Data):** Dependency on manual uploads for cash forecasting introduces a risk of data staleness if not uploaded promptly. (Mitigated by strict HITL review).
* **Risk (Missing Statements):** Oracle occasionally fails to receive all bank statements. TCA must robustly alert Treasury when the statement count mismatches expected volumes.
* **Constraint (FX Data Access):** Historical delays in obtaining FX data access from the Oracle team constrain development. **Mitigation:** TCA employs a multi-source FX strategy. While it aims to integrate with Oracle or GP, it implements the public Frankfurter API as an immediate, reliable fallback to ensure development and operations are never blocked.
* **Assumption:** SERA AI UI components can effectively produce and export the required custom reports.
* **Mitigation:** Currently, there is no viable fallback as the native Oracle categorization solution is deemed inadequate. TCA must act as the primary system of record for this process.
