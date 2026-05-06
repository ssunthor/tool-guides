# Remittance Ingestor (RMI) - Design Document

**Version:** 0.6.0
**Date:** 2026-02-21
**Author(s):** Martin Kecka, Gemini AI
**Category:** Ingestor (I)
**Acronym:** RMI

---

## 1. Introduction & Purpose

The **Remittance Ingestor (RMI)** is a specialized AI-powered backend service designed to automate the first step of the cash application process: understanding unstructured remittance advice. Its core purpose is to monitor a designated email inbox in real-time, extract and interpret remittance documents (PDFs, Excel, images, etc.), and transform them into structured, queryable data.

The RMI employs a robust, multi-stage, failover processing pipeline. It systematically attempts a series of text extraction methods—from Excel parsing to simple OCR and advanced layout-aware models like Marker and Nanonets—to find the best possible text representation of a document.

Once the text is extracted, a multi-stage analysis pipeline uses a series of LLM-powered "specialists" and programmatic rules to:
1.  Identify the customer and refine the true sender.
2.  Extract payment details and invoice line items.
3.  Enrich and programmatically repair the data using database-backed business rules.
4.  Trigger a complex validation and self-correction loop.

This correction loop cross-validates invoice totals against text and email bodies, and triggers multi-pass text and vision-based AI models to repair both incorrect invoice *numbers* and *amounts*. The final, structured data is stored in a PostgreSQL database and exposed via a gRPC API, making it available for downstream systems like the Cash Application Assistant (CAA) to consume.

---

## 2. Architecture & Core Components

The RMI is a self-contained Python application with a clear separation of concerns, designed for reliability and maintainability.

* `remittance_ingestor.py`: The main entry point that bootstraps the application and launches the command-line interface (CLI).
* `cli/`: Contains the `click`-based CLI modules. `rmi_cli.py` defines the main commands for interacting with the service (e.g., starting servers, pollers, processing data).
* `core/ingestion.py`: Manages the live production email polling service, using `core/email_handler.py` to fetch new messages.
* `core/orchestrator.py`: The heart of the application. It manages the end-to-end processing pipeline for a single email, orchestrating the text extraction gauntlet and the subsequent multi-stage analysis.
* `core/queue_manager.py`: A singleton class that manages the in-memory state of reprocessing and active jobs, exposed via the gRPC API.
* `core/llm_interface.py`: Handles all interactions with the AI models. It utilizes a `vLLMClient` that communicates with OpenAI-compatible vLLM endpoints (routed via HAProxy) for high concurrency and performance. It includes an `OllamaClient` compatibility wrapper to maintain seamless integration with the existing pipeline steps.
* `core/pipeline/`: This directory contains the modular, single-responsibility steps that form the processing pipeline, broken into five distinct stages (`a1_...` to `a5_...`).
    * **`a1_extraction/`**: Modules for each text extraction strategy (Excel, OCR, Marker, Nanonets).
    * **`a2_analysis/`**: Modules for core data extraction (customer, payment, invoices, sender).
    * **`a3_enrichment/`**: Modules for programmatic validation and data cleanup (bank details, currency, invoice rules).
    * **`a4_validation_and_correction/`**: The advanced AI-based self-healing loop for amounts and invoice numbers.
    * **`a5_finalization/`**: Modules for final data fusion and classification.
    * `definitions.py`: The single source of truth for the pipeline's logic, defining the exact sequence of all modules.
* `prompts/`: A centralized library of all LLM prompt templates, organized by task.
* `grpc_api/`: Defines and implements the external gRPC interface.
    * `protos/rmi.proto`: The Protobuf definition file. Includes `ListRemittances`, `ReprocessRemittance`, `GetRemittanceDetails`, and `GetProcessingStatus`.
    * `service.py`: The Python implementation of the gRPC server, handling requests and interacting with the `ProcessingQueueManager` and core services.
* **Data Layer**:
    * `core/database/`: Contains the database connection logic (`database.py`) and the SQLAlchemy ORM data models (`models.py`).
    * `core/io_handlers/`: Modules for handling data persistence (`sql_writer.py`, `sql_reader.py`).
    * `core/bank_helper.py` & `core/invoice_helper.py`: **Critical helpers that are database-driven.** They load business rules directly from the `bank_accounts` and `invoice_rules` tables.
* `config/settings.yaml`: The central configuration file.

---

## 3. How an Email is Processed: The Ingestion Pipeline

The RMI's core strength lies in its robust, sequential pipeline that ensures the highest possible success rate for data extraction.

### Step 1: EML Ingestion & Preparation

The process is triggered in one of two ways:
1.  **Production Polling:** The `start-poller` service (`core/ingestion.py`) monitors the MS Graph inbox. When a new email arrives, it is downloaded as an `.eml` file, assigned a unique `processing_id`, and a record is created in the `rmi_processing_results` table with its `message_id`.
2.  **Manual Reprocessing:** A gRPC or CLI call to `reprocess_by_id` triggers the process using an existing `.eml` file from artifact storage.

The `orchestrator` then creates a unique temporary processing directory and parses the `.eml` file to:
* Extract and save the clean email body text to `email_body.txt`.
* Extract, sanitize, and save all relevant attachments (PDF, Excel, images, etc.).
* Identify the primary document to be processed, prioritizing attachments.

### Step 2: The Text Extraction Gauntlet

The orchestrator iterates through the text extraction methods defined in `core/pipeline/definitions.py`. It attempts the first method; only if that method fails (returns an error) does it proceed to the next.

The standard order of attempts is:

1.  **`excel`**: (Highest priority) If the document is an `.xlsx` or `.xls`, this parser directly extracts data using pandas.
2.  **`paged-pdf`**: A specialized, fast extractor for multi-page PDFs using PyMuPDF.
3.  **`ocr`**: Standard Tesseract OCR.
4.  **`marker-ocr`**: Marker model with OCR forcibly enabled.
5.  **`marker`**: The most advanced layout-aware model (Marker).
6.  **`nanonets-transformers`**: (Fallback) A powerful vision-based OCR model. It interfaces with an OpenAI-compatible remote vLLM endpoint hosting the `Nanonets-OCR2-3B` model, routing requests via HAProxy for scalable image-to-text extraction.

The first method to complete this step without a hard error provides the `raw_text` for all subsequent stages.

### Step 3: Stage A2 - Core Data Analysis

The resulting raw text is passed to the A2 analysis pipeline:

* **`identify_customer`**: An LLM call identifies the sending company's name.
* **`repair_vision_customer_name`**: (Fallback) If text-based identification fails, a vision model is used.
* **`refine_sender_email`**: **(New)** If the *initial* sender is internal (e.g., "credit@gtt.net"), this step uses an LLM to scan the email body for the *original external sender* of a forwarded message.
* **`extract_payment_details`**: An LLM extracts header-level info (`total_payment_amount`, `payment_date`, etc.).
* **`extract_invoices`**: An LLM extracts all individual invoice line items.

### Step 4: Stage A3 - Enrichment & Programmatic Repair

The extracted data is cleaned and validated against business rules sourced from the database.

* **`enrich_bank_details`**: Uses `BankHelper` to look up the extracted `bank_account` and `currency` in the `bank_accounts` DB table, enriching the record with the verified bank name.
* **`normalize_currency`**: Standardizes the currency string (e.g., "$") to an ISO code (e.g., "USD").
* **`repair_ocr_invoices`**: Applies a first pass of simple, rule-based string replacements to common OCR errors in invoice numbers (e.g., "1N" -> "IN").
* **`validate_and_repair_invoices`**: **(Critical Step)** Uses `InvoiceHelper` to validate every single invoice number against the `invoice_rules` DB table. It programmatically repairs numbers, finds the matching Business Unit (BU), and flags any that fail validation for the next stage. It utilizes a "Smart Context Repair" mechanism: if standard validation fails, it scans the raw text line of the invalid number to identify and automatically swap in a valid neighboring invoice number that standard extraction may have missed.
* **`normalize_invoice_amounts`**: Based on the matched BU's rules, this step enforces the sign of amounts (e.g., ensures all "CR" prefixed invoices are negative).

### Step 5: Stage A4 - Validation & Correction Loop

This is the AI-powered self-healing stage.

1.  **`validate_totals` (Initial Check)**: The system calculates the sum of extracted invoices and compares it to the `total_payment_amount`. If they don't match, it sets the `trigger_vision_correction` flag.
2.  **Invoice *Number* Repair (Conditional)**: If Step 4 flagged invalid numbers, a multi-pass AI repair is triggered:
    * `repair_text_invoice_number_2ndpass`
    * `repair_vision_invoice_number`
    * `repair_vision_invoice_number_2ndpass`
3.  **Invoice *Amount* Repair (Conditional)**: If the `trigger_vision_correction` flag is set:
    * `cross_validate_total_from_text`: An LLM re-reads the *document text* to find the "ground truth" total, potentially correcting the one from Step 3.
    * `cross_validate_total_from_body`: If still mismatched, an LLM reads the *email body* text as a final cross-check.
    * `repair_vision_invoice_amount`: If still mismatched, a vision model is shown the document and the flawed list and asked to "audit" and fix the invoice *amounts*.
    * `repair_vision_invoice_amount_2ndpass`: If that fails, a second-pass vision prompt is used which trusts the invoice table over any stated total.
4.  **`revalidate_and_repair_numbers_final`**: After potential amount corrections (which can alter the invoice list), this step re-runs the full invoice number validation and repair loop one last time.
5.  **`validate_totals` (Final Check)**: This step runs again to make the *final* decision on the record's status (`SUCCESS`, `PARTIAL_SUCCESS_INVALID_INVOICE`, `FLAGGED_FINAL_REVIEW`, etc.) based on all completed repairs.

### Step 6: Stage A5 - Finalization

* **`fuse_email_body_data`**: **(New)** This step runs to fill in any remaining *header-level* gaps (like `customer_name` or `payment_date`) by prompting an LLM to find them in the email body text.
* **`classify_flagged_email`**: If the pipeline failed to find *any* data, this step classifies the *reason* for the failure (e.G., `FLAGGED_SECURE_PORTAL`, `FLAGGED_OUT_OF_OFFICE`, `FLAGGED_CMD_AUTO_RESPONSE`).
* **`save_artifacts`**: The final, consolidated `RemittanceData` object is saved to the `rmi_processing_results` table, and all generated artifacts are archived to the permanent, date-based artifact storage path.

---

## 4. Key Features & Functionality

* **Live Email Polling**: The `start-poller` service actively monitors the production inbox, ingesting and processing emails in near real-time.
* **gRPC API**: The primary integration point. Allows external systems to:
    * `ListRemittances`: Get paginated, filterable results.
    * `GetRemittanceDetails`: Fetch a single record.
    * `ReprocessRemittance`: Re-queue a failed or old item.
    * `GetProcessingStatus`: **(New)** Monitor the in-memory queue and processing status.
* **Multi-Stage AI Pipeline**: A robust pipeline that combines data extraction, programmatic enrichment from a DB, and multi-modal AI self-correction.
* **Database-Driven Business Logic**: Replaces static CSV files for bank and invoice rules with live PostgreSQL tables, allowing for dynamic updates without redeployment.
* **Test Data Management**: CLI commands (`download-test-data`, `process-test-data`) remain for rapid testing and validation.
* **Reporting & Summarization**: CLI tools to generate a high-level summary (`summary`) and a detailed, flattened CSV report (`convert-to-csv`).

---

## 5. Data Storage

### 5.1. PostgreSQL Database

The database is the source of truth for all structured data and business rules.

* **`rmi_processing_results` table**: The main table for all processed emails. Now includes:
    * `processing_id` (UUID)
    * `message_id` (The MS Graph email ID)
    * `status`, `successful_workflow`, `customer_name`, `total_payment_amount`, `invoices (JSONB)`, `artifacts_path`.
* **`business_units` table**: **(New)** Stores details for each Oracle BU.
* **`bank_accounts` table**: **(New)** The new "bank master" file, storing all account numbers, currencies, bank names, and links to BUs.
* **`invoice_rules` table**: **(New)** Stores all valid invoice/credit memo patterns (prefix, length) for each BU.

### 5.2. File System (Artifact Storage)

All files related to a processing job are stored permanently for auditing and reprocessing.

* **Path Structure**: `/ai/assets/emails/rmi/<YYYY-MM-DD>/<processing_id>/`
    * The `YYYY-MM-DD` subfolder is based on the received date of the email.
* **Contents**: Each `<processing_id>` directory contains:
    * The original source `.eml` file.
    * The primary document that was processed (e.g., `remittance.pdf`).
    * The extracted text file from the successful workflow (e.g., `marker_extracted.txt`).
    * All debug JSON files from all *attempted* workflows (e.g., `final_attempt_ocr_result.json`).

---

## 6. Configuration

The application's behavior is controlled by `config/settings.yaml`.

* **`grpc_server`**: Port and worker settings.
* **`database`**: The DSN connection string.
* **`paths`**: Proto source and generated stub paths.
* **`logging`**: Log level and paths.
* **`msgraph`**: Credentials and settings for the production email polling service.
* **`llm`**: The endpoint configuration for the primary AI model (e.g., `gemma3:27b`). It points to an HAProxy local address (`http://127.0.0.1:8080`) that routes to an OpenAI-compatible vLLM chat completions API, supporting standard request formatting and robust timeout/SSL configurations.
* **`nanonets_llm`**: The endpoint and model name for the Nanonets OCR extractor. It points to the HAProxy router serving the `vllm-nanonets-ocr2-3b` model via the standard OpenAI chat completions format.
* **`output`**: Directories for temp files and permanent artifacts.
* **`company_info`**:
    * `our_banks`: List used by the LLM for entity recognition.
    * `internal_email_domains`: List used by the `refine_sender_email` step to differentiate external senders from internal forwards.

---

## 7. Command-Line Interface (CLI) Usage

```bash
# Initialize the database schema
./remittance_ingestor.py init-db

# Start the production email poller
./remittance_ingestor.py start-poller

# Start the gRPC server to listen for API requests
./remittance_ingestor.py start-grpc-server

# List all available mail folders
./remittance_ingestor.py list-mail-folders

# --- Test Data Commands ---
# Download the 100 most recent emails for local testing
./remittance_ingestor.py download-test-data --count 100

# Process all newly downloaded .eml files not yet in the database
./remittance_ingestor.py process-test-data --process-new

# --- Maintenance Commands ---
# Reprocess a single item that previously failed
./remittance_ingestor.py reprocess-by-id --id "a1b2c3d4-..."

# Generate a high-level summary of the latest test run
./remittance_ingestor.py summary
```

## 8. Future Enhancements

The RMI is currently transitioning from a synchronous, multi-model fallback gauntlet to a unified, vision-first architecture powered by Nanonets and Gemma 4. 

### 8.1. The "Clean Slate" Refactor (Total Purge)
**Objective:** Execute a complete teardown of the legacy extraction codebase. All legacy OCR engines (Marker, Tesseract, Surya, PyMuPDF) and their associated complex downstream math-repair loops (1st/2nd pass logic) will be deleted. The new pipeline will be built from scratch to leverage reasoning models (Gemma 4) over clean data representations.

### 8.2. Phase 2 Architecture: The "Intelligent Router" 
**Objective:** Move away from running sequential fallback extractors. Instead, classify the payload immediately and route it to the most efficient, specialized pipeline track.

**Core Components:**
1.  **The Traffic Controller (Step 1):**
    *   **Input:** Metadata (Sender, Subject, Attachment types) + Email Body.
    *   **Logic:** Checks the `sender_profiles` database table and evaluates the payload presence.
    *   **Action:** Determines the exact `processing_strategy` before any heavy extraction begins.

2.  **Specialized Pipeline Tracks:**
    *   **Track A (The "Noise" Track):** Immediate classification for Auto-responses, Out-of-Office, and Secure Portal notifications. *Cost: Near Zero.*
    *   **Track B (The "Body-Only" Track):** For emails with no valid attachments where the remittance data is in the email body or thread. Bypasses Nanonets completely; passes the raw email string directly to Gemma 4.
    *   **Track C (The "Native" Track):** For `.xlsx` or `.csv` files. Uses native Pandas extraction, directly formatted for Gemma 4.
    *   **Track D (The "Standard Vision" Track):** The default route for standard PDFs/Images. Uses the 3x H100 Nanonets cluster to generate clean HTML tables, followed by a single-pass extraction using Gemma 4 (with `enable_thinking=True` to organically reason through Debit/Credit logic).
    *   **Track E (The "Custom/Garbage" Track):** For notoriously messy senders identified by their profile. Executes hard-coded, customer-specific Gemma 4 prompts or bypasses standard validation rules to handle their specific brand of chaos.

### 8.3. Database & Learning Enhancements (Sender Profiles)
*   **Sender Profile Database:** A new table to store learned preferences and routing rules for specific senders (e.g., `preferred_track='Track E'`, `custom_prompt_id='XYZ'`, `invoice_regex='INV-\d{8}'`). This allows easy "adjusting" of logic for customers who send total garbage.
*   **The "Learning" Feedback Loop:** When an operator manually corrects a record in the Cash Application Assistant (CAA) dashboard, the system captures *why* it failed. This feedback updates the `sender_profile`, ensuring the next email from that customer is routed to the correct custom track automatically.

### 8.4. Advanced Error Handling: Holistic Table Repair (Double-Rip)
**Objective:** Eliminate math errors caused by bad OCR formatting rather than asking the LLM to "fix the math" downstream.

*   **Mechanism:** Adopt the `table_repair.py` logic from the Legal Document Parser (LDP). Before data extraction begins, the system scans the Nanonets HTML output for unclosed table tags (`<tr>`, `<td>`) that bleed across physical page boundaries.
*   **The Double-Rip:** If a broken table spans a page break, RMI will crop the bottom 50% of Page A and the top 50% of Page B, passing the high-res images to Gemma 4 to perfectly reconstruct the missing rows.

### 8.5. Confidence Scoring: Logprobs over Hallucinations
**Objective:** Replace subjective LLM confidence guessing with mathematical certainty.

*   **Mechanism:** When Gemma 4 extracts foundational data (e.g., `customer_name`, `total_payment_amount`), the system will capture the exact token log-probabilities. 
*   **Action:** If the mathematical confidence score of an extracted field falls below a strict threshold (e.g., 85%), the system instantly flags the record for manual review in the CAA, rather than blindly trusting the LLM.

### 8.6. CMD Connector Integration (Guided Validation)
**Objective:** Leverage the central Cash Management Database (CMD) to perform validations of extracted invoice candidates during the RMI extraction phase, improving the context given to the AI repair models.

*   **Mechanism:** RMI will use CMD strictly to *guide* the AI repair loop.
*   **Process Flow:**
    1. During validation, query CMD via gRPC (`validate_invoices`) for all extracted invoice candidates.
    2. If an invoice is *not* found in CMD, the system sets a `trigger_invoice_repair` flag.
    3. The model is prompted with the context that the previously extracted number is invalid in the financial system, forcing it to scrutinize the document specifically for OCR errors on that string.
    4. **Crucially:** If the model insists the number on the page matches the "invalid" extraction, RMI must pass it through unchanged. RMI acts as an objective reader; it must present what the customer actually submitted (even if it's a typo) to the downstream CAA operators for manual investigation.