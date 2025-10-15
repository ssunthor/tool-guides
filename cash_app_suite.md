# High-Level Architectural Design: Automation of Cash Application

## 1. Overview & Goal

The goal of this project is to automate the cash application process, which is currently a highly manual activity. This design outlines a solution built within the SERA ecosystem to automate receipt creation and application in Oracle, based on data from bank statements, remittance advices, and the CMD system.

The architecture is designed to handle the entire workflow, from data ingestion to final posting in Oracle, including the complex business logic and exception handling detailed in the "Receipt creation and application in Oracle Process" flowchart[cite: 1].

## 2. Architectural Pattern: Delegated Autonomy

This solution is a prime candidate for the **Delegated Autonomy** architectural pattern. Instead of being managed directly by the main Sera Orchestrator, the entire business process will be delegated to a new, specialized `(A) Assistant`. This assistant will act as the "AI Accountant," orchestrating a series of deterministic `(C) Connectors` and intelligent `(I) Ingestors` to perform its task. The process will be event-driven, triggered by new payment data rather than a direct user chat command.

## 3. Proposed SERA Modules

The solution will be composed of five new, independent modules within the SERA ecosystem.

### 3.1. Remittance Ingestor (RMI) - (I) Ingestor

* **Category:** `(I) Ingestor`
* **Purpose:** To monitor the `credit@gtt.net` mailbox, parse incoming emails, and extract structured data from unstructured remittance advice attachments (e.g., PDF, Excel, jpg).
* **Responsibilities:**
    * Connect to the email server to read new messages.
    * Download and identify remittance advice attachments.
    * Use a powerful comprehension model from the AI Factory (e.g., `gemma3:27b`) for OCR and structured data extraction, converting documents into a queryable JSON object.
    * Store the structured remittance data in a dedicated `remittances` table in the shared PostgreSQL database for the assistant to query.

### 3.2. Bank Statement Ingestor (BSI) - (I) Ingestor

* **Category:** `(I) Ingestor`
* **Purpose:** To automatically ingest the daily bank statement and load each payment transaction into the shared database, triggering the application process.
* **Responsibilities:**
    * Run on a daily schedule to connect to the bank data source (e.g., SFTP, API) and download the new statement.
    * Parse the structured file to extract individual payment records (Payer Name, Amount, Date, Reference Info).
    * Insert each payment as a new row into a `payments` table in PostgreSQL with a status of "unprocessed."

### 3.3. Oracle Connector (ORC) - (C) Connector

* **Category:** `(C) Connector`
* **Purpose:** To provide a safe, deterministic, and LLM-free gRPC API for all necessary actions within Oracle, directly mapping to the steps in the process flowchart[cite: 1].
* **gRPC Methods:**
    * `CreateReceiptBatch(batch_details)`: Initiates a new batch in Oracle as the first step in the process[cite: 1].
    * `AddPaymentToBatch(batch_id, payment_details)`: Adds a specific payment record to an existing batch.
    * `FindOpenItems(customer_id)`: Returns all open items for a customer, including invoices and Credit Memos (CMs).
    * `ValidateItem(item_id)`: Checks if an item (invoice/CM) is valid and returns its status, amount, and due date.
    * `ApplyPaymentToItems(batch_id, payment_id, item_applications)`: Applies a payment to one or more specified invoices/CMs within a batch.
    * `PostBatch(batch_id)`: Finalizes and posts the batch, making the transactions permanent in Oracle[cite: 1].

### 3.4. CMD Connector (CMC) - (C) Connector

* **Category:** `(C) Connector`
* **Purpose:** To act as a simple, LLM-free bridge to the CMD system for client information lookups.
* **gRPC Methods:**
    * `GetClientInfo(client_name)`: Retrieves the unique client ID/BAN based on the payer name found on the bank statement.

### 3.5. Cash Application Assistant (CAA) - (A) Assistant

* **Category:** `(A) Assistant`
* **Purpose:** This is the **brain** of the operation. ]It is a specialized assistant that embodies the complex decision-making logic of a cash applications specialist as depicted in the flowchart[cite: 1]. It orchestrates the entire process from start to finish.
* **Responsibilities:**
    * Use a powerful reasoning model (`gemma3:27b`) to analyze incoming payments and execute a multi-step plan.
    * Orchestrate gRPC calls to the ORC and CMC to interact with the source systems.
    * Query the PostgreSQL database for ingested remittance and payment data.
    * Execute a complex decision tree to handle various scenarios, including perfect matches and exceptions.
    * Identify specific variance types noted in the flowchart, such as **Bank fee**, **Double payment**, **Overpayment**, and **Bad invoice detail**[cite: 1].
    * Manage the Human-in-the-Loop (HITL) workflow by pausing, escalating for manual validation with clear findings and options, and resuming upon receiving a decision.

## 4. End-to-End Automated Workflow

The workflow is designed to handle both the "happy path" and the numerous exceptions identified in the business process.

1.  **Trigger:** The **BSI** ingests a new payment and calls the **CAA** to begin the application process.

2.  **Step 1: Setup & Information Gathering**
    * The CAA receives the payment details.
    * It makes its first call to `ORC.CreateReceiptBatch()` to initialize the process in Oracle.
    * It queries the PostgreSQL database for any matching remittance data from the **RMI**.
    * It calls `CMC.GetClientInfo()` to resolve the payer name to a unique customer ID.

3.  **Step 2: Automated Analysis (The "Happy Path")**
    * A single, matching remittance is found in the database. The total amount perfectly matches the payment.
    * The CAA validates each invoice listed on the remittance by calling `ORC.ValidateItem()`.
    * If all invoices are valid and open, the CAA proceeds directly to the execution step.

4.  **Step 3: Exception Analysis & HITL Escalation (The "Unhappy Path")**
    * The CAA's reasoning engine analyzes any discrepancy based on the flowchart's logic.
    * **No Remittance Found:** The CAA calls `ORC.FindOpenItems()`. If a single invoice matches the payment amount exactly, it proceeds. Otherwise, it escalates to the HITL with a list of open invoices as suggested applications.
    * **Amount Mismatch:** If the remittance total does not match the payment, the CAA investigates the cause:
        * It checks for known **bank fees** that could account for the difference.
        * It queries its own logs for recent, identical payments to flag a potential **double payment**.
        * For any other **overpayment** or **underpayment**, it escalates to the HITL for a decision.

5.  **Step 4: Execution and Completion**
    * Once a valid plan is confirmed (either automatically or via human approval), the CAA makes the final calls.
    * It calls `ORC.ApplyPaymentToItems()` with the precise, validated mappings.
    * It calls `ORC.PostBatch()` to finalize the transactions in Oracle.
    * The entire transaction, including the reasoning and any human decisions, is recorded in an audit log in PostgreSQL.

## 5. Human-in-the-Loop (HITL) Integration

The multiple "Manual validation" steps in the flowchart necessitate a robust HITL workflow[cite: 1]. This is not a separate module but a core capability of the **CAA**.

* **Pause and Escalate:** When the CAA cannot make a decision with high confidence, it will pause the workflow and persist its current state.
* **Structured Request:** It will send a structured request to a human review queue (e.g., a dedicated MS Teams channel, Jira, or a simple web UI). This request will contain:
    * All gathered context (payment info, remittance data, open invoices).
    * The specific problem identified (e.g., "Overpayment of $50.25 detected").
    * A set of clear, actionable options for the human operator (e.g., "Apply to oldest invoice," "Leave on account as unapplied cash," "Reject payment").
* **Resume on Response:** The CAA will wait for a structured response via an API call from t