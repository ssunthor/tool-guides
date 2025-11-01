# Remittance Ingestor (RMI) - Design Document

**Version:** 0.4.0
**Date:** 2025-10-09
**Author(s):** Gemini AI
**Category:** Ingestor (I)
**Acronym:** RMI

---

## 1. Introduction & Purpose

The **Remittance Ingestor (RMI)** is a specialized AI-powered backend service designed to automate the first step of the cash application process: understanding unstructured remittance advice. Its core purpose is to monitor a designated email inbox, extract and interpret remittance documents (PDFs, text, images, etc.), and transform them into structured, queryable data.

The RMI employs a robust, multi-stage, failover processing pipeline. It systematically attempts a series of text extraction methods—from simple text parsing to advanced layout-aware and OCR models—to find the best possible text representation of a document. Once the text is extracted, a shared analysis pipeline uses a series of LLM-powered "specialists" to identify the customer, extract payment details, and list individual invoices.

A key feature of the RMI is its automated validation and self-correction loop. After the initial text-based extraction, the system cross-validates the sum of extracted invoices against the stated total payment. If a discrepancy is found, it automatically triggers a multi-pass vision-based correction pipeline, using an advanced vision model to re-analyze the original document image and repair the flawed data.

The final, structured data is stored in a PostgreSQL database and exposed via a gRPC API, making it available for downstream systems like the Cash Application Assistant (CAA) to consume and act upon.

---

## 2. Architecture & Core Components

The RMI is a self-contained Python application with a clear separation of concerns, designed for reliability and maintainability.

-   `remittance_ingestor.py`: The main entry point that bootstraps the application and launches the command-line interface (CLI).
-   `cli/`: Contains the `click`-based CLI modules. `rmi_cli.py` defines the main commands for interacting with the service (e.g., starting servers, processing data), while `summary_cli.py` provides reporting capabilities.
-   `core/orchestrator.py`: The heart of the application. It manages the end-to-end processing pipeline for a single email (`.eml` file), orchestrating both the text extraction gauntlet and the subsequent shared analysis steps.
-   `core/pipeline/`: This directory contains the modular, single-responsibility steps that form the processing pipeline.
    -   **Text Extraction Modules**: (`extract_text_ocr.py`, `extract_text_marker.py`, etc.) Each module represents a different strategy for converting a source document into raw text.
    -   **Analysis Modules**: (`identify_customer.py`, `extract_invoices.py`, etc.) Each module performs a discrete analysis task on the extracted text.
    -   `definitions.py`: The single source of truth for the pipeline's logic, defining the exact sequence of text extraction methods to try and the shared analysis steps that follow.
-   `prompts/`: A centralized library of all LLM prompt templates. Each file contains prompts tailored for a specific task (e.g., `invoices.py`, `payment.py`), ensuring consistency and easy management of the AI's instructions.
-   `grpc_api/`: Defines and implements the external gRPC interface.
    -   `protos/rmi.proto`: The Protobuf definition file for all services, messages, and data structures.
    -   `service.py`: The Python implementation of the gRPC server, handling requests and interacting with the core services.
-   **Data Layer**:
    -   `core/database/`: Contains the database connection logic (`database.py`) and the SQLAlchemy ORM data models (`models.py`).
    -   `core/io_handlers/`: A set of modules for handling data persistence (`sql_writer.py`, `sql_reader.py`, `assets_writer.py`).
-   `config/settings.yaml`: The central configuration file for the entire module.

---

## 3. How an Email is Processed: The Ingestion Pipeline

The RMI's core strength lies in its robust, sequential pipeline that ensures the highest possible success rate for data extraction. The entire process is orchestrated by `core/orchestrator.py`.

### Step 1: EML Ingestion & Preparation

The process begins with a single `.eml` file. The orchestrator first creates a unique temporary processing directory for it. It then parses the `.eml` file to:

-   Extract and save the clean email body text to `email_body.txt`.
-   Extract, sanitize the filenames of, and save all attachments (PDF, Excel, images, etc.) into the temporary directory.
-   Identify the primary document to be processed, prioritizing attachments like PDFs and images over the email body. If no attachments are found, the email body itself is used as the source document.

### Step 2: The Text Extraction Gauntlet

This is a sequential, failover-driven process designed to find the optimal text representation of the primary document. The orchestrator iterates through the text extraction methods defined in `core/pipeline/definitions.py`. It attempts the first method; only if that method fails to produce a `SUCCESS` status does it proceed to the next.

The standard order of attempts is:

1.  **`paged-pdf`**: A specialized, fast extractor that handles multi-page PDFs by extracting text from each page separately. This workflow is skipped if the document is not a multi-page PDF.
2.  **`ocr`**: A standard Optical Character Recognition (OCR) process using Tesseract. It converts any PDF page or image into raw text. This is effective for simple, clean documents.
3.  **`marker-ocr`**: An advanced method that uses the Marker model with OCR forcibly enabled. This is excellent for documents where text is embedded within images or is otherwise un-extractable.
4.  **`marker`**: The most advanced layout-aware model. It parses the document's structure (like a human would) to produce a clean markdown output. This is the most powerful method for complex layouts with columns and tables but is also the most resource-intensive.

The first method to successfully complete the entire downstream analysis pipeline (Steps 3-5) is declared the **winner**, and its results are used.

### Step 3: The Shared Analysis Pipeline

Once a text extraction method succeeds, the resulting raw text is passed through a standardized sequence of analysis steps, also defined in `definitions.py`:

-   **Identify Customer**: An LLM call identifies the sending company's name from the document's header or logo text.
-   **Repair Vision Customer Name (Fallback)**: If the text-based method fails, a vision model is used to look at the document image to find the customer's name or logo.
-   **Extract Payment Details**: An LLM extracts header-level information like `total_payment_amount`, `payment_date`, and `payment_ref_id`. For multi-page documents, it intelligently uses text from the first and last pages.
-   **Enrich Bank Details**: If a destination bank account number is found, this step cross-references it with the `bank_master.csv` file to pull in clean, verified bank details.
-   **Normalize Currency**: Standardizes the extracted currency string (e.g., "$", "Pound Sterling") to its proper ISO 4217 code (e.g., "USD", "GBP") using a lookup map (`currency_map.csv`) with an LLM fallback for unknown currencies.
-   **Extract Invoices**: This is the most critical extraction step. An LLM is prompted to parse the document text and extract all individual invoice line items, carefully handling credit notes and negative values based on a set of critical rules. For multi-page documents, this step is run on each page, and the results are aggregated.
-   **Validate Totals**: The system programmatically calculates the sum of all extracted invoice amounts. It then compares this sum to the `total_payment_amount` extracted in the payment details step.
    -   If the totals match, the status is set to `SUCCESS`.
    -   If they do not match, a discrepancy is recorded, and a `trigger_vision_correction` flag is set to `True`.

### Step 4: The Vision Correction Loop (Conditional)

This powerful self-correction mechanism is only triggered if the `trigger_vision_correction` flag was set in the previous step.

1.  **First Pass Repair**: The orchestrator calls the `repair_vision_invoice_amount` step. This sends the original document image and the flawed invoice list to a vision-capable LLM. The prompt explains the discrepancy and instructs the model to act as an auditor, re-examining the image to produce a corrected invoice list that sums to the stated total payment.
    -   If the corrected list now matches the total, the status is updated to `SUCCESS_VISION_CORRECTED`.
    -   If it still fails, the status is set to `FLAGGED_VISION_FAILED`.
2.  **Second Pass Repair**: If the first pass failed, the orchestrator triggers the `repair_vision_invoice_amount_2ndpass` step. This step uses a different, more robust prompt. It tells the LLM that the stated total itself might be wrong and that the invoice table in the image is the single source of truth. This forces the model to ignore potentially misleading summary fields and rebuild the invoice list purely from the visual data.
    -   If this pass succeeds in creating a valid list, the status is set to `SUCCESS_VISION_CORRECTED_2ND_PASS`.
    -   If it also fails, the document is marked for final human review (`FLAGGED_FINAL_REVIEW`).

### Step 5: Final Classification & Storage

-   **Flagged Email Classification**: If the pipeline fails early on because no invoice or payment data could be found, a final `classify_flagged_email` step is run. It analyzes the email body to assign a more specific status, such as `FLAGGED_SECURE_PORTAL`, `FLAGGED_OUT_OF_OFFICE`, or `FLAGGED_FOLLOW_UP`.
-   **Save Final Result**: The final, consolidated `RemittanceData` object is saved to the `rmi_processing_results` table in PostgreSQL.
-   **Archive Artifacts**: All files generated during the single successful workflow (e.g., `marker_extracted.txt`, debug JSON files, etc.) are copied from the temporary directory to a permanent, persistent artifacts location for auditing and reprocessing.

---

## 4. Key Features & Functionality

-   **gRPC API**: The primary integration point for other services. It allows external systems to list remittances, get details for a specific record, and trigger reprocessing.
-   **Automated Pipeline**: Fully automates the process of reading an email, analyzing its contents, and storing structured results without human intervention.
-   **Multi-Modal AI**: Uses both text-based and vision-based AI models to maximize extraction accuracy and provide self-correction capabilities.
-   **Test Data Management**: Includes CLI commands to easily download a recent set of emails (`download-test-data`) and process them in bulk (`process-test-data`), enabling rapid testing and validation.
-   **Reprocessing Capabilities**: Any processed item can be re-run through the latest version of the pipeline by its unique ID (`reprocess-by-id`). Includes specialized scripts for fixing common historical issues (`fix-multipage-pdfs`).
-   **Reporting & Summarization**: CLI tools are provided to generate a high-level summary of processing results (`summary`) and a detailed, flattened CSV report suitable for business users (`convert-to-csv`).

---

## 5. Data Storage

### 5.1. PostgreSQL Database

The database is the source of truth for all structured data extracted by the RMI.

-   **`rmi_processing_results` table**: The main table where every processed email has a record. Key columns include:
    -   `processing_id`: A unique UUID for each record.
    -   `source_eml_file`: The original filename of the EML.
    -   `status`: The final processing outcome (e.g., `SUCCESS`, `FLAGGED_FOR_REVIEW`).
    -   `successful_workflow`: The name of the text extraction method that succeeded (e.g., `marker-ocr`).
    -   `customer_name`, `total_payment_amount`, `payment_date`, `currency`.
    -   `invoices`: A `JSONB` column containing the list of extracted invoice numbers and amounts.
    -   `amount_discrepancy`: The calculated difference between the stated total and the sum of invoices.
    -   `error_message`: A detailed error message if the process was flagged.
    -   `artifacts_path`: The absolute path to the directory containing all associated files.

### 5.2. File System (Artifact Storage)

All files related to a processing job are stored permanently for auditing and reprocessing.

-   **Path Structure**: `/ai/assets/emails/rmi/<YYYY-MM-DD>/<processing_id>/`
    -   The `YYYY-MM-DD` subfolder is based on the received date of the email.
-   **Contents**: Each `<processing_id>` directory contains:
    -   The original source `.eml` file.
    -   The primary document that was processed (e.g., `remittance.pdf`).
    -   The extracted text file from the successful workflow (e.g., `marker_extracted.txt`).
    -   Debug JSON files from various pipeline steps.

---

## 6. Configuration

The application's behavior is controlled by `config/settings.yaml`.

-   **`grpc_server`**: Port and worker settings for the gRPC API.
-   **`database`**: The DSN connection string for the PostgreSQL database.
-   **`paths`**: Defines source and destination paths, including the critical `generated_stubs_path` for shared gRPC stubs.
-   **`logging`**: Log level, directory, and filenames.
-   **`msgraph`**: Credentials and settings for connecting to the Microsoft Graph API to read emails.
-   **`llm`**: The endpoint, model name (`gemma3:27b`), and timeout settings for the AI Factory's LLM service.
-   **`output`**: Defines base directories for output, temporary files, and permanent artifacts.
-   **`company_info`**: Contains paths to master data files (`bank_master.csv`) and a list of `our_banks` used for entity recognition.

---

## 7. Command-Line Interface (CLI) Usage

The RMI is managed via a set of `click` commands.

[bash]
# Initialize the database schema
./remittance_ingestor.py init-db

# Download the 100 most recent emails for local testing
./remittance_ingestor.py download-test-data --count 100

# Process all newly downloaded .eml files not yet in the database
./remittance_ingestor.py process-test-data --process-new

# Reprocess a single item that previously failed
./remittance_ingestor.py reprocess-by-id --id "a1b2c3d4-..."

# Start the gRPC server to listen for API requests
./remittance_ingestor.py start-grpc-server

# Generate a high-level summary of the latest processing run
./remittance_ingestor.py summary
[/bash]

## 8. Future Enhancements

-   **CAA Integration & Feedback Loop**: Develop a mechanism for the RMI to receive corrected data from the Cash Application Assistant. This feedback could be used to flag common errors or potentially fine-tune prompts for specific customers.
-   **Data Fusion (Email Body + Attachments)**: Implement a final "Data Fusion" step in the pipeline. If a primary document (like a PDF) is missing a critical piece of information (e.g., currency, payment date), this step would analyze the email body text to find the missing data and merge it into the final result, creating a more complete record.
-   **Context-Aware Currency Disambiguation**: Enhance the `normalize_currency` step. Instead of defaulting ambiguous symbols like `$` to `USD`, use other context (e.g., customer's country, if known) to make a more intelligent default (e.g., `CAD` for Canadian customers).
-   **Human-in-the-Loop (HITL) UI**: Develop a simple web interface for business users to review items with a `FLAGGED_FOR_REVIEW` status. This UI would display the extracted data alongside the original document image, allowing users to quickly validate, correct, and approve the results.
-   **Profile-Driven Intelligent Pipeline (Sender-Specific Workflows)**: This is the most critical planned enhancement to move the RMI from a "brute-force gauntlet" model to an intelligent, learning system. The current design wastefully processes every email through the same blind, multi-step pipeline, which is inefficient for known senders and resource-intensive for complex documents.
    -   **Core Problem:** The current pipeline runs every extraction method (`ocr`, `marker`, `nanonets`) and every repair step on all documents, regardless of sender. This "one-size-fits-all" approach is slow and does not learn from past successes or failures.
    -   **Proposed Solution:**
        1.  **Create a "Sender Profile" Database:** A new table (e.g., `sender_profiles`) will be created, keyed by the sender's email address or domain. This profile will store an optimized workflow configuration, such as:
            -   `primary_extractor`: The *one* extractor known to work best (e.g., `excel`, `marker-ocr`).
            -   `invoice_regex`: A sender-specific regex for highly accurate invoice validation, replacing the generic validator.
            -   `skip_repairs`: A list of repair steps to *skip* for this sender (e.g., `["repair_vision_customer_name"]`).
            -   `trust_level`: A flag (e.g., `high`, `low`) to determine if validation is even needed.
        2.  **Redesign Orchestrator Logic:** The `orchestrator` will be modified to first look up a profile for the incoming `sender_email`.
            -   **Profile Found:** The orchestrator runs a fast, optimized workflow defined by the profile (e.g., "Run `excel` extractor, validate with `invoice_regex`, skip all repairs").
            -   **No Profile Found:** The orchestrator runs the *current* "gauntlet" pipeline to discover the best processing method.
        3.  **Implement a "Learning Feedback Loop":** A Human-in-the-Loop (HITL) review UI will be created for `FLAGGED_FOR_REVIEW` items. When an operator corrects a document from an "Unknown Sender," they will not only fix the data but also *create* the Sender Profile (e.g., "For this sender, the correct extractor was `marker` and the regex is `INV-\d{8}`"). This profile is then saved, ensuring the *next* email from that sender is processed instantly and accurately.