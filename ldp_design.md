# Legal Document Parser (LDP) - Design Document

**Version:** 0.2.0
**Date:** 2026-05-08
**Author(s):** Martin Kecka, Gemini AI
**Category:** Processor (P)
**Acronym:** LDP

---

## 1. Introduction & Purpose

The **Legal Document Parser (LDP)** is a heavy-duty, asynchronous AI processing engine designed to completely digitize and analyze complex legal agreements (MSAs, Amendments, SOWs, etc.). Serving as the "Grinder" of the Legal Knowledge Base (LKB) suite, its primary responsibility is to transform unstructured, multi-page PDF documents into clean, standardized Markdown and extract critical foundational metadata.

Because legal documents are often dense, poorly scanned, and contain complex nested tables, the LDP abandons synchronous processing in favor of a robust **Celery and Redis-backed worker architecture**. It rips documents into individual pages, processes them concurrently via a hybrid approach (native extraction + vision-based OCR), and seamlessly stitches them back together. 

To guarantee data integrity, the LDP has evolved into a **Self-Healing Pipeline**. It employs sophisticated auditing and repair mechanisms utilizing state-of-the-art vision models (Gemma 4 with Nanonets fallbacks) to fix broken OCR outputs, repair HTML tables that span across page breaks, and enforce strict semantic Markdown hierarchies using specialized models (Gemma 4-E4B). Finally, it utilizes a "Sandwich" LLM prompting strategy and mathematical logprob analysis to extract and resolve contracting parties against the company's master database (CMC).

---

## 2. Architecture & Core Components

The LDP is built as a highly scalable, decoupled service designed to handle hundreds of concurrent long-running PDF processing tasks without blocking the main orchestrator.

* `legal_document_parser.py`: The main entry point that bootstraps the environment and launches the CLI.
* `cli/ldp_cli.py`: The `click`-based CLI managing database operations, pipeline testing, evaluation scripts, targeted repairs, and starting the gRPC server.
* `grpc_api/server.py`: The external interface (`ldp.proto`) allowing the broader LKB Orchestrator to submit documents (`SubmitDocument`), check processing status (`GetJobStatus`), and manage job states (`ReprocessDocument`, `CancelDocument`, `ReprocessMetadata`).
* `core/worker_celery.py`: The distributed task queue engine. It defines the master workflow (`process_document_master`), concurrent hybrid page processing (`process_hybrid_page_task`), and the massive stitching/finalization phase (`finalize_document`).
* `core/orchestrator.py`: Handles initial job ingestion, saving files to the artifact directory, and queuing jobs into the pending database state for Celery Beat to dispatch.
* `core/extraction/`: 
    * `hybrid_parser.py`: The core page processor implementing "Universal Interleaved Segmentation", routing content dynamically between native PyMuPDF extraction and Nanonets Vision OCR.
    * `nanonets_parser.py`: The primary engine for converting PDF crops to images and calling the Nanonets OCR2-3B model.
    * `gemma_extractor.py`: Manages the 4-pass LLM metadata extraction (Category, Detailed Category, Customer, Date) utilizing the Sandwich Method and logprob-based confidence scoring.
    * *Legacy Parsers*: Contains `fitz_parser.py`, `marker_parser.py`, and `surya_parser.py` (maintained for fallback or benchmarking purposes).
* `core/repair/`:
    * `page_validator.py`: Executes the VLM-based "Audit & Repair" loop on individual pages using Gemma 4.
    * `table_repair.py`: Contains the "Double-Rip" logic to fix broken HTML tags spanning page boundaries.
    * `format_repair.py`: Applies Gemma 4-E4B to enforce proper Markdown header hierarchies natively.
    * `header_repair.py`: A regex-based fallback tool to clean up artifact headers and TOC noise.
* `core/enhancing/cmd_customer.py`: Resolves the extracted customer names or emails against the external `cmd_connector` (CMC) via gRPC and historical lookup logic.
* `core/database/`: SQLAlchemy ORM mapping (`models.py`) tracking `DocumentJob` states, review status, extracted data, and deep processing statistics (including the `audit` JSONB log).

---

## 3. How a Document is Processed: The Pipeline

The LDP processes heavy legal PDFs using a scatter-gather (map-reduce) pattern managed by Celery, employing a multi-stage strategy that prioritizes accuracy and self-correction over raw speed.

### Step 1: Submission & Dispatch
1. **Ingestion:** A document is submitted via the gRPC API or CLI. The `orchestrator` saves the raw PDF to the `input/` directory and creates a `DocumentJob` in PostgreSQL with a `pending` status.
2. **Dispatch:** A Celery Beat task (`dispatch_pending_jobs`) polls the database. If the cluster has capacity (e.g., `< 3` active jobs), it assigns the pending job to a worker and updates the status to `processing`.

### Step 2: Universal Interleaved Segmentation (Scatter)
Instead of treating a PDF page as a single image, the `process_document_master` dispatches sub-tasks that analyze the internal topology of the PDF page:
1. **Native Detection**: PyMuPDF scans the page for native vector tables. If a table contains > 10 rows and passes a structural sanity check, it is extracted natively as raw HTML.
2. **Vision Masking**: These native tables are replaced in the validation text with a protective tag: `[TRUSTED_NATIVE_TABLE_MASKED]`.
3. **Vision OCR**: The remaining elements (text bands, small tables, complex/broken matrices) are cropped as images and sent to **Nanonets OCR2-3B** for vision-based extraction.
4. **Initial Assembly**: The native HTML and vision-extracted text are interleaved back together chronologically to form the page topology map.

### Step 3: VLM-Based Quality Audit & Targeted Repair
Once a page is initially extracted, it enters a localized "Self-Healing" loop (`audit_and_repair_pages`):
1. **The Auditor**: An image of the full page (along with context from the previous page) is sent to **Gemma 4 (31B)** with a `VALIDATOR_PROMPT`. The model compares the image to the masked Markdown.
2. **Evaluation**: The model classifies the page structure and assigns an accuracy score.
3. **Targeted Visual Repair**: If discrepancies are found (e.g., hallucinated text loops, missing rows, broken columns), the auditor generates a specific `suggested_repair_instruction`. The flawed Markdown and the image are sent *back* to Gemma 4 to rewrite and fix the specific error, all while preserving the `[TRUSTED_NATIVE_TABLE_MASKED]` tags.
4. **Failsafe**: If Gemma 4 fails or times out, it gracefully retains the original Nanonets output.

### Step 4: Stitching & Holistic Table Repair (Gather - The "Double-Rip")
Once all pages successfully complete their local audits, the `finalize_document` callback is triggered.
1. **Stitching:** The individual page Markdowns are concatenated.
2. **Detection:** The `repair_markdown_tables` function uses regex to scan for unclosed `<tr>` or `<table>` tags that bleed across physical page breaks.
3. **The "Double-Rip" Repair:**
    * If a broken table boundary is found between Page X and Page X+1, the system crops the **bottom 50%** of Page X and the **top 50%** of Page X+1 as high-res images.
    * It prompts the **Gemma 4 (31B)** vision model to perfectly reconstruct just the missing `<tr>` rows, bridging the gap.
    * The hallucinated or broken tags are surgically cleaned, and the freshly generated rows are stitched back into the global document.

### Step 5: Semantic Formatting (E4B Pass)
To prepare the document for Vector Database chunking, the structural hierarchy must be perfect.
1. The repaired text is optionally passed through **Gemma-4-E4B**, a model specifically tuned for formatting.
2. It evaluates the natural outline of the contract and injects proper Markdown headers (`#`, `##`, `###`) without altering the underlying legal text or tables, ensuring clean RAG ingestion downstream.

### Step 6: Intelligent Metadata Extraction
The pristine Markdown is passed to the LLM (`gemma-3-27b-it`) via the `gemma_extractor`.
1. **The Sandwich Method:** If the document exceeds 50,000 characters, the LDP dynamically slices it, keeping the top 75% and bottom 25% (where legal metadata lives) while gutting the middle to save context limits and reduce latency.
2. **4-Pass Extraction:**
    * Extracts broad `category` (e.g., MSA, SOW).
    * Extracts `detailed_category`.
    * Extracts `customer` and related fields (Contact Name, Email) using forced JSON mode. It explicitly ignores internal GTT/MegaPath entities via hardcoded blocking lists.
    * Extracts `date`.
3. **Logprob Confidence Scoring:** The system captures the exact token log-probabilities to generate a mathematical confidence score (0-100%) for the customer extraction, automatically flagging low-confidence hits.
4. **Fallback Healing:** If the model accidentally extracts an internal entity, the system automatically fires a `CUSTOMER_FALLBACK_PROMPT` to self-correct.

### Step 7: CMD Resolution & Finalization
1. **Entity Resolution:** The extracted customer email (or name) is fired against the `cmc_client` (via gRPC). It utilizes fuzzy matching to find the exact official CMD Client Group ID.
2. **Historical Disambiguation:** If the name is ambiguous, it cross-references the `extracted_customer_contact_name` against historical successful jobs in the Postgres DB to confidently map the entity.
3. **Review Routing:** If a CMD match is found, the job is flagged `auto_resolved`. If not, it is flagged `unassigned` for Human-In-The-Loop (HITL) review.
4. **Artifact Storage:** The final Markdown, detailed JSON stats, deep VLM audit logs, and database records are committed, and temporary images are wiped.

---

## 4. Key Features & Functionality

* **Universal Interleaved Segmentation:** Maximizes fidelity by utilizing PyMuPDF for perfect native digital tables while deferring scanned stamps, text, and signatures to Vision OCR.
* **VLM Fast Audit (Self-Healing):** Employs a high-parameter vision model (Gemma 4) to act as an automated QA auditor, flagging and fixing hallucination loops or missed clauses before they reach the database.
* **Double-Rip Table Repair:** A highly specialized computer vision technique that reconstructs complex HTML tables that are physically severed by page breaks.
* **Semantic Formatting (E4B):** Dedicated pass to enforce strict H1/H2/H3 header hierarchies, making the output immediately optimized for semantic RAG chunking.
* **Asynchronous Queueing:** Massive PDFs don't block the API. Celery handles retries, timeouts, and parallel page execution seamlessly.
* **Logprob Confidence Scoring:** Replaces subjective AI "confidence" with objective mathematical probability based on token generation likelihood, improving hit-rate visibility.
* **Internal Entity Blocking:** Hardcoded logic ensures that internal service providers (e.g., GTT, MegaPath, Interoute) are never mistakenly identified as the client.
* **Smart History Resolution:** If the LLM extracts an ambiguous name, the LDP checks the database history for the specific *Customer Contact Name* to disambiguate the company.

---

## 5. Data Storage

### 5.1. PostgreSQL Database
The `document_jobs` table acts as the source of truth for the processing lifecycle:
* **Core Info:** `id` (UUID), `filename`, `status`, `review_status`.
* **Extracted Data:** `extracted_customer`, `extracted_category`, `extracted_date`, `extracted_email`.
* **Resolution Data:** `cmd_client_group_name`, `cmd_client_id`.
* **System Stats:** JSONB `stats` column, JSONB `audit` column (containing the page-by-page VLM evaluations), `extracted_confidence_score`, `error_message`.

### 5.2. File System (Artifact Storage)
* **Input Directory:** `/ai/legal_document_parser/input/` (Raw PDFs).
* **Processed Directory:** `/ai/legal_document_parser/output/<job_id>/`
    * `<filename>.md`: The fully stitched, formatted, and repaired Markdown.
    * `<filename>_stats.json`: Granular breakdown of extraction confidence, runtimes per stage, and heuristic OCR quality metrics.
    * `<filename>_validation_stats.json`: The raw JSON output of the VLM auditor detailing discrepancies and repairs applied.

---

## 6. Command-Line Interface (CLI) Usage

The LDP includes a powerful suite of CLI tools for local testing, evaluation, targeted repairs, and pipeline management.

```bash
# Start the gRPC server to listen for Orchestrator submissions
./legal_document_parser.py start-grpc-server

# Run the standard production pipeline locally on a specific file
./legal_document_parser.py production_pipeline --file "Action_2021_MSA.pdf" --debug

# Re-run Gemma metadata extraction logic on existing markdown files (useful for prompt tweaking)
./legal_document_parser.py refresh_metadata --pass customer --debug

# Scan existing markdowns, detect broken tables, and trigger the Double-Rip Vision API repair loop
./legal_document_parser.py repair_tables

# Scan existing markdowns and repair artifact headers and TOC noise (Dry Run)
./legal_document_parser.py repair_headers

# Apply header repairs directly to the files
./legal_document_parser.py repair_headers --commit

# Re-queue failed or specific jobs back into the Celery cluster
./legal_document_parser.py reprocess_db --status failed

# Evaluate extraction differences against the database (Dry-Run prompt testing)
./legal_document_parser.py test_prompt --field customer --limit 50

# View a high-density CLI table of all processed documents and their CMD resolutions
./legal_document_parser.py inspect_compact --limit 20
```

## 7. Architectural Evolution & Future Enhancements

### 7.1. Future Enhancement: Native Office Format Support
**Objective:** Bypass the OCR and vision pipelines entirely when native digital formats are ingested to maximize throughput and perfectly preserve document structure.
* **Implementation Plan:** Develop and integrate `excel_parser.py` (utilizing Pandas/OpenPyXL) and `word_parser.py` (utilizing Python-Docx) to convert `.xlsx` and `.docx` files directly into strict, RAG-compliant Markdown. 
* **Routing Integration:** This logic will be integrated directly into the `process_document_master` task. If a native Office document is detected during ingestion, the system will instantly parse the underlying XML tree and route the output straight to the Semantic Formatting (E4B) and Metadata Extraction stages. This bypasses the computationally expensive PDF ripping, vision OCR, and VLM audit stages altogether, significantly reducing processing latency and compute costs for these specific file types.
