# Legal Document Parser (LDP) - Design Document

**Version:** 1.0.0
**Date:** 2026-04-14
**Author(s):** Martin Kecka, Gemini AI
**Category:** Processor (P)
**Acronym:** LDP

---

## 1. Introduction & Purpose

The **Legal Document Parser (LDP)** is a heavy-duty, asynchronous AI processing engine designed to completely digitize and analyze complex legal agreements (MSAs, Amendments, SOWs, etc.). Serving as the "Grinder" of the Legal Knowledge Base (LKB) suite, its primary responsibility is to transform unstructured, multi-page PDF documents into clean, standardized Markdown and extract critical foundational metadata.

Because legal documents are often dense, poorly scanned, and contain complex nested tables, the LDP abandons synchronous processing in favor of a robust **Celery and Redis-backed worker architecture**. It rips documents into individual pages, processes them concurrently via vision-based OCR (Nanonets), and seamlessly stitches them back together. 

To guarantee data integrity, the LDP employs a sophisticated **Holistic Table Repair** mechanism (the "Double-Rip" strategy) using state-of-the-art vision models (Gemma 4 with Nanonets fallbacks) to fix broken HTML tables that span across page breaks. Finally, it utilizes a "Sandwich" LLM prompting strategy to extract and resolve contracting parties against the company's master database (CMC).

---

## 2. Architecture & Core Components

The LDP is built as a highly scalable, decoupled service designed to handle hundreds of concurrent long-running PDF processing tasks without blocking the main orchestrator.

* `legal_document_parser.py`: The main entry point that bootstraps the environment and launches the CLI.
* `cli/ldp_cli.py`: The `click`-based CLI managing database operations, pipeline testing, evaluation scripts, and starting the gRPC server.
* `grpc_api/server.py`: The external interface (`ldp.proto`) allowing the broader LKB Orchestrator to submit documents (`SubmitDocument`), check processing status (`GetJobStatus`), and manage job states (`ReprocessDocument`, `CancelDocument`).
* `core/worker_celery.py`: The distributed task queue engine. It defines the master workflow (`process_document_master`), concurrent page OCR (`process_single_page`), and the stitching/finalization phase (`finalize_document`).
* `core/orchestrator.py`: Handles initial job ingestion, saving files to the artifact directory, and queuing jobs into the pending database state for Celery Beat to dispatch.
* `core/extraction/`: 
    * `nanonets_parser.py`: The primary engine for converting PDF pages to images and calling the Nanonets vision model for OCR.
    * `gemma_extractor.py`: Manages the 4-pass LLM metadata extraction (Category, Detailed Category, Customer, Date) and includes logprob-based confidence scoring.
    * *Legacy Parsers*: Contains `fitz_parser.py`, `marker_parser.py`, and `surya_parser.py` (currently being phased out in favor of purely vision-based approaches).
* `core/repair/table_repair.py`: Contains the "Double-Rip" logic. It detects broken HTML tags spanning page boundaries and crops the top/bottom halves of the original PDF pages, querying Gemma 4 to reconstruct the broken rows.
* `core/enhancing/cmd_customer.py`: Resolves the extracted customer names or emails against the external `cmd_connector` (CMC) via gRPC to find the official `cmd_group_id`.
* `core/database/`: SQLAlchemy ORM mapping (`models.py`) tracking `DocumentJob` states, review status, extracted data, and deep processing statistics.

---

## 3. How a Document is Processed: The Pipeline

The LDP processes heavy legal PDFs using a scatter-gather (map-reduce) pattern managed by Celery.

### Step 1: Submission & Dispatch
1. **Ingestion:** A document is submitted via the gRPC API or CLI. The `orchestrator` saves the raw PDF to the `input/` directory and creates a `DocumentJob` in PostgreSQL with a `pending` status.
2. **Dispatch:** A Celery Beat task (`dispatch_pending_jobs`) polls the database. If the cluster has capacity (e.g., `< 3` active jobs), it assigns the pending job to a worker and updates the status to `processing`.

### Step 2: Ripping & Concurrent OCR (Scatter)
1. **Initialization:** The `process_document_master` task rips the entire PDF into individual high-res PNG images and saves them to a temporary directory.
2. **Concurrent Execution:** For every page, a `process_single_page` Celery sub-task is fired in parallel. 
3. **Vision OCR:** Each task sends its respective image to the Nanonets OCR2-3B endpoint. The model reads the page naturally, capturing text, converting tables to HTML `<tr>`/`<td>` tags, and outputting Markdown.

### Step 3: Stitching & Holistic Table Repair (Gather)
Once all pages complete, the `finalize_document` callback is triggered.
1. **Stitching:** The individual page Markdowns are concatenated.
2. **Detection:** The `repair_markdown_tables` function uses regex to scan for unclosed `<tr>` or `<table>` tags that bleed across page breaks.
3. **The "Double-Rip" Repair:**
    * If a broken table is found on Page X, the system crops the **bottom 50%** of Page X and the **top 50%** of Page X+1 as high-res images.
    * It prompts the **Gemma 4 (31B)** vision model to perfectly reconstruct just the missing `<tr>` rows.
    * If Gemma 4 fails or times out, it seamlessly falls back to the Nanonets vision model.
    * The hallucinated or broken tags are cleaned, and the freshly generated rows are smartly stitched back into the Markdown document.

### Step 4: Intelligent Metadata Extraction
The repaired, pristine Markdown is passed to the LLM (Gemma-3-27b) via the `gemma_extractor`.
1. **The Sandwich Method:** If the document exceeds 50,000 characters, the LDP dynamically slices it, keeping the top 75% and bottom 25% (where legal metadata lives) while gutting the middle to save context limits.
2. **4-Pass Extraction:**
    * Extracts broad `category` (e.g., MSA, SOW).
    * Extracts `detailed_category`.
    * Extracts `customer` and related fields (Contact Name, Email) using forced JSON mode, explicitly ignoring internal GTT/MegaPath entities. It captures **token logprobs** to assign a mathematical confidence score to the extraction.
    * Extracts `date`.

### Step 5: CMD Resolution & Finalization
1. **Entity Resolution:** The extracted customer email (or name) is fired against the `cmc_client` (via gRPC). It utilizes fuzzy matching and historical DB lookups to find the exact official CMD Client Group ID.
2. **Review Routing:** If a CMD match is found, the job is flagged `auto_resolved`. If not, it is flagged `unassigned` for Human-In-The-Loop (HITL) review.
3. **Quality Metrics:** The system calculates `dictionary_ratio`, `table_density`, and `special_char_ratio` to grade the cleanliness of the OCR output.
4. **Artifact Storage:** The final Markdown, detailed JSON stats, and database records are committed, and temporary images are wiped.

---

## 4. Key Features & Functionality

* **Asynchronous Queueing:** Massive PDFs don't block the API. Celery handles retries, timeouts, and parallel page processing.
* **Double-Rip Table Repair:** A highly specialized vision-fallback system that ensures HTML tables are perfectly formed even when split awkwardly across physical pages.
* **Logprob Confidence Scoring:** Instead of guessing if the AI is hallucinating, the LDP calculates the exact token probability of the extracted customer name, flagging low-confidence extractions automatically.
* **Internal Entity Blocking:** Hardcoded logic ensures that internal service providers (e.g., GTT, MegaPath, Interoute) are never mistakenly identified as the client.
* **Smart History Resolution:** If the LLM extracts an ambiguous name, the LDP checks the database history for the specific *Customer Contact Name* to disambiguate the company.

---

## 5. Data Storage

### 5.1. PostgreSQL Database
The `document_jobs` table acts as the source of truth for the processing lifecycle:
* **Core Info:** `id` (UUID), `filename`, `status`, `review_status`.
* **Extracted Data:** `extracted_customer`, `extracted_category`, `extracted_date`, `extracted_email`.
* **Resolution Data:** `cmd_client_group_name`, `cmd_client_id`.
* **System Stats:** JSONB `stats` column, `extracted_confidence_score`, `error_message`.

### 5.2. File System (Artifact Storage)
* **Input Directory:** `/ai/legal_document_parser/input/` (Raw PDFs).
* **Processed Directory:** `/ai/legal_document_parser/output/<job_id>/`
    * `content_nanonets.md`: The fully stitched and repaired Markdown.
    * `<filename>_stats.json`: Granular breakdown of extraction confidence, runtime per page, and heuristic OCR quality metrics.

---

## 6. Command-Line Interface (CLI) Usage

The LDP includes a powerful suite of CLI tools for local testing, evaluation, and pipeline management.

```bash
# Start the gRPC server to listen for Orchestrator submissions
./legal_document_parser.py start-grpc-server

# Run the standard production pipeline locally on a specific file
./legal_document_parser.py production_pipeline --file "Action_2021_MSA.pdf" --debug

# Re-run Gemma metadata extraction logic on existing markdown files (useful for prompt tweaking)
./legal_document_parser.py refresh_metadata --pass customer --debug

# Scan existing markdowns, detect broken tables, and trigger the Vision API repair loop
./legal_document_parser.py repair_tables

# Evaluate extraction differences against the database (Dry-Run prompt testing)
./legal_document_parser.py test_prompt --field customer --limit 50

# View a high-density CLI table of all processed documents and their CMD resolutions
./legal_document_parser.py inspect_compact --limit 20
```

## 7. Future Enhancements

While the LDP currently maintains legacy parser code (Marker, Surya, Fitz, Tesseract), the architecture is actively transitioning toward a highly consolidated vision.

### 7.1. Model Consolidation
**Objective:** Phase out heavy local layout models (Marker, Surya) completely.
* **End State:** The pipeline will rely exclusively on **Nanonets** for raw PDF text/layout extraction and **Gemma 4** for targeted visual repairs and metadata analysis.

### 7.2. Native Office Format Support
**Objective:** Bypass OCR entirely when native formats are available.
* Implementation of `excel_parser.py` (via Pandas) and `word_parser.py` (via Python-Docx) to convert `.xlsx` and `.docx` directly to Markdown. This will be integrated into the `process_document_master` routing logic before defaulting to the PDF/Nanonets pipeline.
