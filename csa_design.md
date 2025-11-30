# Customer Support Assistant (CSA) - Design Document

**Version:** 0.1.0
**Date:** 2025-11-20
**Author(s):** Gemini AI, Martin Kecka
**Category:** Assistant (A)
**Acronym:** CSA

---

## 1. Introduction & Purpose 💡

The **Customer Support Assistant (CSA)** is an AI-enhanced orchestration module designed to automate the initial triage, categorization, and client linking of inbound customer inquiries sent to the Customer Success Center. Its primary purpose is to address the high volume of repetitive, manual tasks by providing a modern, integrated solution for case load management, thereby meeting the objective of automating 80-90% of repeatable tasks.

The CSA operates by monitoring the dedicated support inbox, processing each email, and executing a staged categorization pipeline to determine the customer's intent, priority, and required action. It exposes a **Human-in-the-Loop (HITL)** user interface for the CSC team to manage, correct, and provide the essential feedback necessary for continuous model improvement. 

### Core Objectives (Phase 1)

1.  **Triage Automation:** Replace the manual, round-robin email sorting with an automated, standardized categorization process.
2.  **Client Linking:** Identify and link the incoming email address to an authoritative client record using the **CMD Connector (CMC)**.
3.  **Intent Categorization:** Accurately classify the customer request into one of the explicit categories defined by the CSC team (e.g., Renewal, Billing, Disconnect).
4.  **UI & Feedback Loop:** Deliver a user-facing portal for monitoring, intervention, and data correction to facilitate rule generation.

---

### 2. Architecture & Core Components 🧱

The CSA is a standalone Python application adhering to the SERA **Assistant (A)** pattern. It is built on a high-performance Python/FastAPI stack, utilizing **gRPC** for internal communication and **PostgreSQL** for state management and configuration storage. The core AI capabilities are provided by an internal LLM service (AI Factory).

---

#### Data Sources / Inputs:

* **MS Graph API:** Provides new emails from the production support inbox (`CustomerSuccessCenter@gtt.net`) via the `EmailPoller`.
* **CMD Connector (CMC):** Accessed via gRPC for authoritative client lookup, retrieval of associated metadata (`GetClientByEmail`), and **Customer Segment (SMB/Enterprise)** data.
* **LLM Service (AI Factory):** Provides the core intelligence for intent classification and sentiment analysis (using `gemma3:27b`).
* **PostgreSQL Database:**
    * **Owns:** `support_requests` (state machine), `categorization_rules` (dynamically generated Python rules).
    * **Reads:** Customer Account Data (from external CRM/Envision DX sources, potentially mirrored/accessed via CMC/dedicated connector).

#### Core Components

* `csa_main.py`: The application entry point, handling configuration loading, logging setup, launching the `click` CLI, the background Poller, and the FastAPI/gRPC servers.
* `cli/csa_cli.py`: Defines all external **CLI commands** for operational control and maintenance.
* `core/ingestion/email_poller.py`: The deterministic poller service. Manages OAuth2 token acquisition for MS Graph, monitors the support inbox, and downloads emails, creating a **NEW** record in the DB and storing the raw `.eml` file to artifact storage.
* `core/orchestrator.py`: The high-level **Rule Engine**. Manages the state machine, retrieves the processing record from the DB, and executes the sequence of pipeline steps.
* `core/pipeline/`: Contains the modular, single-responsibility steps that form the processing pipeline.
    * `a1_prepare_email.py`: Extracts clean text from the `.eml` file, handling attachments and text encoding issues. **Includes PII Redaction Layer for text sent to LLM/Database.**
    * `a2_client_linking.py`: Implements the core **CMC gRPC call** to retrieve `cmd_client_id` and **`customer_segment`**.
    * `a3_categorization_logic.py`: Manages the LLM interaction and rule-based classification.
    * `a4_finalization.py`: Commits results, updates status, and flags exceptions.
* `grpc_api/`: The external communication layer.
    * `protos/csa.proto`: Defines the gRPC service and all data transfer objects (DTOs).
    * `services.py`: Implements the `CustomerSupportServiceServicer`, providing logic to fulfill RPC requests from downstream systems (e.g., querying categorization results).

---

### 3. Core Functionality & Process Flow (Phase 1) ⚙️

The **Triage Orchestrator** drives the process through a defined sequence of steps to achieve reliable categorization. **Phase 1 focuses exclusively on accurate categorization, client linking, and data enrichment to support downstream automation modules.**

---

#### 3.1. Orchestration Flow (4 Key Logical Steps)

The orchestration operates as a series of checks and executions, designed to maximize deterministic, non-LLM rule application before engaging the more costly AI factory.

1.  **Check Rules/Link Client:**
    * Executes the `CategorizeByPredefinedRules` (checking the `categorization_rules` table).
    * Executes the `LinkClientBySenderEmail` rule (CMC gRPC call to retrieve `cmd_client_id` and **`customer_segment`**).
2.  **Primary Classification:** Executes the `LLM_AnalyzeIntent` rule against the constrained category list.
3.  **Priority/Sentiment:** Executes the `LLM_AnalyzeSentiment` rule.
4.  **Finalization/Audit:** Executes the `FinalizeTriageResult` rule, committing the final status and audit trail.

#### 3.2. Intent Classification Categories

The LLM prompt is strictly constrained by the categories derived from the initial business analysis, including the high-volume campaign responses:

| Type | Categories |
| :--- | :--- |
| **Contract/Renewal** | Renewal, Auto Renew Inquiry, Contract Info, Disconnects |
| **Financial/Billing**| Billing General, Account Detail |
| **System/Service** | MAC (Move/Add/Change), RMA Equipment, Services Delivery Escalation, EV Training/Changes/New user requests |
| **Escalation/Other**| NOC Ticket/Escalation, CSR Request |
| **Campaign Response**| **Outbound Campaign Responses** |

#### 3.3. Categorization Pipeline Details (Revised Logic)

The rule engine executes rules sequentially. If a rule is **conclusive** (sets a terminal or diagnostic status), the engine stops. If it is **inconclusive** but changes the state (e.g., links a client), the engine re-evaluates the rules for the new state.

| Step | Rule | Component | Decision Logic |
| :--- | :--- | :--- | :--- |
| **A** | `PrepareAndRedactEmail` | `a1_prepare_email.py` | Extracts clean text. **Redacts PII/CPNI** before storing redacted text in DB or sending to LLM. |
| **B** | `CategorizeByPredefinedRules` | DB/Orchestrator | Checks `categorization_rules` for deterministic Python/Regex matches (generated by HITL). **Conclusive Output:** Sets category, status `COMPLETE_AWAITING_ACTION`, stops. **Inconclusive:** Proceeds to C. |
| **C** | `LinkClientAndSegment` | CMC gRPC Call | Retrieves `cmd_client_id` and **`customer_segment`** (required for reporting). **Inconclusive Output:** Sets client data, proceeds to D. |
| **D** | `LLM_AnalyzeIntent` | LLM Service | Calls `gemma3:27b` to classify intent. **Conclusive Output:** Sets category, status `COMPLETE_AWAITING_ACTION` (if confidence > 85%), proceeds to E. **Inconclusive:** Proceeds to E. |
| **E** | `LLM_AnalyzeSentiment`| LLM Service | Assesses urgency/tone to set the `priority_flag` (P1/P2/P3). **Inconclusive Output:** Sets priority, proceeds to F. |
| **F** | `FinalizeTriageResult` | Orchestrator | Commits the record and executes the **Diagnostic Fallback** (See 3.4) if status is not yet conclusive. **Conclusive Output:** Sets a specific `hitl_...` status. |

#### 3.4. Diagnostic Fallback (New Section, Reusing CAA Pattern)

If the categorization pipeline reaches the `FinalizeTriageResult` step (F) without setting a conclusive, automated status (e.g., `COMPLETE_AWAITING_ACTION`), the engine initiates a diagnostic sequence, adopting the CAA's pattern of providing a specific reason for **Human-in-the-Loop (HITL)** intervention. 

| Diagnosis Check | Logic | Conclusive Output |
| :--- | :--- | :--- |
| **Client Link Check** | If `cmd_client_id` is null after all linking attempts (Step C). | `hitl_client_not_found` |
| **Confidence Check** | If `LLM_AnalyzeIntent` resulted in a `Confidence Score < 85%`. | `hitl_low_confidence` |
| **Ambiguity Check** | If the LLM returned multiple plausible categories with similar high confidence scores. | `hitl_ambiguous_intent` |
| **Default Fallback** | All checks passed, but no conclusive rule was met (e.g., for complex, custom requests). | `hitl_triage_needed` |

---

### 4. Data Flows and Configuration 💾

#### Data Flow

1.  **Ingestion:** `Customer Success Inbox` $\rightarrow$ `EmailPoller` $\rightarrow$ **CSA DB (status: NEW)** $\rightarrow$ *Artifact Storage*
2.  **Processing:** **CSA DB (NEW)** $\rightarrow$ `Orchestrator` $\rightarrow$ **CMC (gRPC)** $\rightarrow$ `LLM Service (gRPC)` $\rightarrow$ **CSA DB (status: COMPLETE/FLAGGED)**
3.  **Consumption:** `CSR UI (FastAPI)` $\rightarrow$ **CSA DB** $\rightarrow$ *Audit/Feedback Loop*

#### Audit & Artifact Storage

All processing data is stored permanently for auditing and reprocessing. **The raw `.eml` file containing PII is stored separately from the processed, redacted text.**

* **Path Structure**: `/ai/assets/emails/csa/<YYYY-MM-DD>/<request_id>/`
* **Contents**: Original `.eml` file (secure storage), **Redacted** `email_body.txt`, and full LLM/Orchestrator debug JSON payloads.

---

### 5. User Interface (React SPA) 💻

The CSA UI is a Single Page Application (React/FastAPI) essential for CSR workflow management, **directly reusing the Monitoring, Intervention, and Feedback components established in the CAA design.**

#### 5.1. Monitoring & Triage Dashboard (Enhanced with Diagnosis)

* **Queue Management:** Displays a unified queue of all support requests with filterable views for the **Diagnostic States** (e.g., `hitl_client_not_found`, `hitl_low_confidence`, `hitl_triage_needed`), ensuring CSRs address issues based on the specific failure reason.
* **Intervention:** Includes UI buttons to trigger **Reprocessing** for individual requests or bulk items based on filter (directly mirroring CAA's bulk action capabilities).
* **KPIs:** Real-time display of key performance indicators: **Automation Rate (%)**, **Average Response Time (hrs)**, and **Error Rate (%)**.

#### 5.2. Detailed Review & Correction (The Feedback Loop)

This is the **Human-in-the-Loop (HITL)** interface where ground truth is provided, **directly replicating the CAA's correction and learning mechanism.**

1.  **Review Panel:** CSR views the raw email text (or redacted text), the AI's predicted fields, the confidence score, and the **Diagnostic Reason** (`hitl_...`).
2.  **Correction:** The CSR manually corrects and saves the authoritative values for: **`cmd_client_id`**, **`primary_category`**, and **`priority_flag`** (crucially, selecting a value from the constrained list in 3.2).
3.  **Feedback Submission (Core Reuse):** Upon saving the correction, the record status is set to **`HITL_CORRECTED`**. This explicit action triggers the **Model Learning Service (7.1)**, ensuring a clean and auditable feedback path identical to the CAA's successful model.

---

## 6. Command-Line Interface (CLI) Usage

The `csa_cli.py` module uses `click` for development and operational control.

```bash
# Initialize the database schema and load initial rules
./csa_main.py init-db

# Start the background service to continuously poll the support inbox
./csa_main.py start-poller

# Start the gRPC server to expose the categorization results to other systems
./csa_main.py start-grpc-server

# Force processing of all emails currently marked as 'NEW'
./csa_main.py process-all-new

# Test the client lookup logic for a known email ID
./csa_main.py test-cmc-link --id "a1b2c3d4-..."
```

### 7. Future Enhancements (Phase 2 Roadmap) 🚀

The Phase 1 output (categorization and linking) serves as the input for these Phase 2 automation modules, shifting the focus from triage to action execution and intelligence scaling. **This will be achieved by integrating new modules that this core Triage Assistant (CSA) can call.**

---

#### 7.1. Model Learning Service (Rule/Prompt Generation)

* **Action:** Implement a dedicated **Model Learning Service** that monitors the CSA DB for **`HITL_CORRECTED`** records.
* **Logic:** The service uses the corrected data (raw text + true category) and the **Gemma LLM** to analyze patterns and generate outputs designed to improve future performance:
    * **Dynamic Rule Generation:** Generate new, high-confidence, deterministic **Python/Regex rules** (e.g., matching a sender domain or specific keyword sequence to a category) and store them in the **`categorization_rules`** table, reducing future reliance on the LLM.
    * **Prompt Refinement:** Analyze common failure modes and suggest adjustments to the core LLM prompt template used by the orchestrator.

#### 7.2. Action Execution: CMD Ticket Creation

* **Action:** Implement a `TicketActionService` that monitors the CSA DB for `COMPLETE_AWAITING_ACTION` records where the category demands a new case (e.g., `Billing`, `Technical Support`).
* **Integration:** Call the existing **CMC** gRPC method (`CMC.CreateTicket(client_id, category, summary, customer_segment)`) to log the case in the ticketing system.
* **Result:** Update the request status to **`ACTION_CMD_TICKET_CREATED`**.

#### 7.3. Action Execution: Automated Response

* **Action:** For simple query categories (e.g., `Auto Renew Inquiry`, `Account Detail`), engage a content retrieval service (EnvisionDX/CRM data) to fetch the answer.
* **Output:** The service drafts a professional, templated response using the retrieved data, marks the request as **`ACTION_AUTO_RESPONDED`**, and sends the email (adhering strictly to the **Communication Standards** constraint).

#### 7.4. Action Execution: ServiceNow & External Module Integration (New)

* **Action:** Implement a dedicated routing module to handle escalations or service requests that reside in external systems, such as **ServiceNow**.
* **Integration:** This module will monitor categorized tickets (e.g., `NOC Ticket/Escalation`) and trigger the appropriate external API call to create or update a case in the external system.
* **Result:** Update the request status to **`ACTION_EXTERNAL_CASE_CREATED`**.


##Draft info ->
Mapped Components for Reuse 🗺️The design for the Customer Support Assistant (CSA) will leverage the following concepts and functional components established in the CAA design:CAA Component / FeaturePurpose in CAACSA Mapping / ReuseRule Engine (Sec 3)State machine execution; Conclusive vs. Inconclusive rules; Fallback logic.Direct Reuse: The CSA's core/orchestrator.py will follow the exact same state machine logic (e.g., stopping on conclusive status, re-evaluating on state change).Fallback & Diagnosis (Sec 3)Sets a specific, descriptive hitl_... status (e.g., hitl_rmi_mismatch) instead of a generic hitl_needed.Critical Reuse: The CSA's FinalizeTriageResult (Step D in 3.3) must set specific diagnostic statuses (e.g., hitl_low_confidence, hitl_client_link_fail) instead of just FLAGGED_FOR_REVIEW.HITL UI (Sec 6)Monitoring, detailed drill-downs, Intervention (reprocessing triggers).Direct Reuse: The CSA UI must adopt this look and feel, replacing transaction/invoice views with Support Request views.Feedback Submission (Sec 6)System for users to submit feedback and trigger AI analysis.Direct Reuse: This is the core of the CSA's Model Learning Service (7.1). The HITL correction in the UI (HITL_CORRECTED status) must feed into this module, mimicking CAA's approach.Background Tasks (Sec 3/5)Proactive linking/matching to populate a cache.Conceptual Reuse: CAA uses background tasks (linking_service.py) to build cache (email_client_link). CSA can use this pattern for proactive data enrichment (e.g., pre-fetching contract details for highly likely renewal requests).