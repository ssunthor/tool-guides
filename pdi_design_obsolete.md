# PDF Ingestor (PDI) - Full Design & Development Plan

**Version:** 0.2
**Date:** 21-Aug-2025
**Status:** Draft

---

## 1. Overview

This document provides the complete design specification for the PDF Ingestor (PDI). The PDI is a standalone Python application for processing PDF documents and ingesting their content and AI-generated metadata into the AI Assistant's data stores.

The primary goals for this project are:

* **Build a Self-Contained Ingestor:** The PDI will be built by porting the proven, source-agnostic components from the Confluence Ingestor (CFI) into a new, independent application.
* **Implement an Integrated Processing Pipeline:** Based on the planned enhancements for the CFI, the PDI will feature a unified, single-pass workflow. For each document, all processing steps—extraction, chunking, summarization, semantic analysis, HyDE generation, and embedding—will be executed sequentially. This ensures that every piece of data stored is complete and ready for RAG solutions from the moment of ingestion.
* **Evaluate Dual Parsing Strategies:** The ingestor will be architected to support two configurable content extraction methods—a library-based parser (PyMuPDF) and a vision-based AI model (`nemoretriever-parse`)—to allow for a thorough comparison of their performance and quality.

---

## 2. Features

* **Source Handling:** Scans a configurable source file directory for PDF documents.
* **Change Detection:** Implements a full incremental sync strategy based on file modification timestamps and content hashes.
* **Dual-Strategy Content Extraction:** Extracts text, tables, and images from PDF files via a configurable strategy (library or vision).
* **Category Management:** Automatically derives a content hierarchy from the source directory's subdirectory structure.
* **Asset Extraction:** Extracts embedded images and saves them to shared storage.
* **Unified Enrichment Pipeline:** Executes a complete, single-pass enrichment process for all new or updated documents:
    * **Chunking:** Splits extracted text into manageable chunks.
    * **LLM Summarization:** Generates two-pass (detailed and brief) summaries for the full document and each chunk.
    * **Table Summarization:** Creates natural language summaries for all extracted tables.
    * **LLM Semantic Analysis:** Generates structured metadata (`document_type`, `user_intent`, `key_entities`, `actionability_score`) for each chunk.
    * **Hypothetical Question Generation (HyDE):** Creates relevant questions that each chunk can answer.
    * **Image Analysis (Optional):** Generates keywords and descriptions for extracted images if the `--vision` flag is used.
    * **Embedding:** Generates vector embeddings for chunks (using an enriched text representation) and for the overall document summary.
* **Robust Storage:** Persists all extracted content and generated metadata consistently across PostgreSQL, ChromaDB, and Shared Asset Storage.
* **Error Handling:** Logs failed ingestion items to a dedicated database table for review.

---

## 3. Architecture & Core Components

### 3.1. Foundational Design: The New Orchestrator Core

The PDI will be built around a new, highly modular orchestrator. The `core/pdf_orchestrator.py` module is not a direct port of the CFI's existing `processor.py`. Instead, it represents the implementation of the planned "Redesign the `processor.py`" enhancement. It will be built from the ground up with a focus on a fully incremental, resilient, and manageable workflow that executes the entire processing pipeline in a single, coherent pass for each document.

### 3.2. Component Porting & Creation Plan

The standalone nature of the PDI will be achieved by porting and creating the necessary components.

| Component / File | Action | Implementation Notes |
| :--- | :--- | :--- |
| `pdf_ingestor.py` | Create New | New entry point. CLI argument logic will be adapted from the CFI, removing post-processing flags and focusing on ingestion control. |
| `core/pdf_orchestrator.py` | Create New | Main workflow orchestrator, implementing the new single-pass pipeline design. |
| `core/pdf_processor.py` | Create New | Contains the switchable logic for both PyMuPDF and `nemoretriever-parse` extraction strategies. |
| `sql_writer.py` | Port from CFI | Direct copy. The database schema is source-agnostic. |
| `vdb_writer.py` | Port from CFI | Direct copy. The vector database schema is source-agnostic. |
| `database/models.py` | Port from CFI | Direct copy. The schema supports a `source_type` of 'pdf'. |
| `embedder.py` | Port from CFI | Direct copy. Embedding logic is independent of the source text. |
| `llm_summarizer.py` | Port from CFI | Direct copy. Operates on extracted text. |
| `qa_generator.py` | Port from CFI | Direct copy. HyDE generation is source-agnostic. |
| `table_summarizer.py` | Port from CFI | Direct copy. Works on the standard structured JSON for tables. |
| `llm_analyzer.py` | Port from CFI | Direct copy. Semantic analysis is source-agnostic. |
| `text_chunker.py` | Port from CFI | Direct copy. Text splitting is universal. |
| `asset_writer.py` | Port from CFI | Direct copy. The logic for storing files is generic. |
| `prompts/*.py` | Port from CFI | All existing prompts operate on extracted text and are directly reusable. |
| `requirements.txt` | Create New | Will include all ported dependencies from CFI, plus new ones like `PyMuPDF`. |

### 3.2. Folder Structure

/ai/
│
└── pdf_ingestor/                
├── config/
│   ├── settings.yaml
│   └── vdb_schema.yaml
├── core/
│   ├── asset_writer.py
│   ├── embedder.py
│   ├── llm_analyzer.py
│   ├── llm_summarizer.py
│   ├── qa_generator.py
│   ├── sql_writer.py
│   ├── table_summarizer.py
│   ├── text_chunker.py
│   ├── vdb_writer.py
│   ├── pdf_processor.py      # <-- NEW: Handles all PDF parsing logic
│   └── pdf_orchestrator.py   # <-- NEW: Orchestrates the PDF ingestion workflow
├── database/
│   └── models.py
├── prompts/
│   ├── pdf_ingestor_prompts.py
│   └── post_processing_prompts.py
├── utils/
│   └── logger.py
├── bootstrap_helper.py
├── pdf_ingestor.py      # <-- NEW: Entry point for PDI
└── requirements.txt

---

## 4. Integrated Process Flow

The `PdfOrchestrator` will execute the following integrated pipeline for each PDF file:

1.  **Scan & Detect:** Identify a new or modified PDF file in the `source_directory`.
2.  **Extract:** The `PdfProcessor` is called to extract all raw content (text, images, tables as JSON) using the configured strategy (library or vision).
3.  **Store Assets:** Extracted images are immediately saved to permanent storage via the `AssetWriter`.
4.  **Delete Old Data:** If the document is being reprocessed, all associated records (chunks, summaries, questions, etc.) are deleted from PostgreSQL and VDB.
5.  **Chunk Text:** The raw extracted text is split into chunks by the `TextChunker`.
6.  **Process Chunks in a Loop:** For each chunk of text:
    a. **Generate LLM Summary:** The `LLMSummarizer` creates a detailed and a brief summary for the chunk.
    b. **Generate Semantic Analysis:** The `LLMAnalyzer` creates the structured JSON with `document_type`, `user_intent`, etc.
    c. **Generate HyDE Questions:** The `QAGenerator` creates a set of hypothetical questions the chunk can answer.
    d. **Summarize Tables:** If the chunk contains table placeholders, the `TableSummarizer` generates natural language summaries for the corresponding JSON table data.
    e. **Prepare for Embedding:** The text for the vector embedding is constructed. This enriched text combines the original chunk content with its summaries and table summaries to create a dense representation of its meaning.
    f. **Generate Embedding:** The `Embedder` creates the vector embedding from the enriched text.
    g. **Aggregate:** The chunk's original text, its embedding, and all its generated metadata (summaries, analysis, questions) are aggregated into a final payload.
7.  **Store All Data:** The `SqlWriter` and `VdbWriter` are called once with the complete data package for the document.
    * **PostgreSQL:** The `sources_metadata` record is written, along with all `source_chunks` records (which now include columns for the LLM analysis JSON), and all associated `llm_summaries` and `generated_questions`.
    * **Vector DB:** The embeddings for each chunk are written to the main collection. The document-level summary is embedded and written to the summary collection.
8.  **Finalize:** The original PDF is marked as processed (by hash) if hash change, pdf is reprocessed automatically, unless --force-reprocess is triggered

---

## 5. Data Model & Storage Schema 📚

To ensure full independence from the Confluence Ingestor (CFI), the PDI will operate on its own dedicated database tables and vector database collections. This approach guarantees complete data isolation.

* **PostgreSQL:**
    The PDI will use a **table prefix** defined in the configuration to create a new, separate set of tables. While it will reuse the same source-agnostic data models (`database/models.py`), all PDI data will be written to tables prefixed with, for example, `pdf_`. This results in tables like `pdf_sources_metadata` and `pdf_source_chunks`, keeping them entirely separate from CFI data.

* **Vector DB (ChromaDB):**
    Similarly, the PDI will write to a **dedicated collection name** specified in its configuration (e.g., `pdf_main` and `pdf_summaries`). This ensures that vector embeddings from PDF documents are not mixed with those from any other source.

This strategy guarantees that the PDI's data is completely isolated, preventing any potential overlap while still allowing for the reuse of proven schema definitions.


---

## 6. Configuration (settings.yaml)

The `settings.yaml` file will be self-contained for the PDI. Post-processing sections are now part of the main `processing` block and are enabled by default.

```yaml
# PDF Ingestor Source & Parsing Configuration
pdf_ingestor:
  source_directory: "/ai/assets/pdfs"
  file_extensions: [".pdf"]

pdf_processing:
  strategy: "vision"
  library_parser: {}
  vision_parser:
    client_type: "nim"
    nim:
      base_url: "[https://aif-nim-nemoretriever-parse.slo.gt-t.net](https://aif-nim-nemoretriever-parse.slo.gt-t.net)"
      model: "nvidia/nemoretriever-parse-v0.2-beta"
      api_key_env_var: null
      request_timeout: 300
    pdf_to_image_dpi: 300

# Unified Processing and Enrichment Configuration
processing:
  chunking: { ... }
  embedding: { ... }
  llm_summarizer: { ... }
  vision: { ... } # For extracted images
  qa_generation: { ... }
  table_summarization: { ... }
  llm_analyzer: { ... }
  # ... other processing settings

storage:
  asset_base_path: "/ai/assets"

  postgresql:
    db_name: "ai_assistant_db"
    user_env_var: "DB_USER"
    password_env_var: "DB_PASS"
    host: "localhost"
    port: 5432
    table_prefix: "pdf_" # <-- NEW: Defines the prefix for all PDI tables.

  vdb:
    client_type: "chromadb"
    chromadb:
      host: "localhost"
      port: 8000
      collection_name: "pdf_main" # <-- UPDATED: PDI's unique collection name.
      summary_collection_name: "pdf_summaries" # <-- UPDATED: PDI's unique summary collection.

logging: { ... }
```

---

## 7. Command-Line Interface (CLI)
The CLI is simplified to reflect the integrated pipeline.

# Runs the full ingestion and enrichment pipeline.
python main_pdf_ingestor.py

# Forces reprocessing of all files, ignoring change detection.
python main_pdf_ingestor.py --force-reprocess

# Specifies a custom settings file.
python main_pdf_ingestor.py --settings /path/to/custom_settings.yaml

# Processes only a limited number of files for testing.
python main_pdf_ingestor.py --limit 10

# (Optional developer flags) Can be added to bypass specific, 
# time-consuming enrichment steps during debugging.
python main_pdf_ingestor.py --skip-llm-analysis --skip-hyde

---

## 8. Key Dependencies

* **Ported:** `SQLAlchemy`, `psycopg2-binary`, `chromadb`, `requests`, `PyYAML`, `python-dotenv`, `langchain`.
* **New:** `PyMuPDF`, `Pillow`.

---

## 9. Future Enhancements & Porting Plan

The PDI will maintain feature parity with the CFI by porting enhancements as they are developed.

* **Redesign the `processor.py`:**
    * **PDI Implementation:** This is a foundational requirement for the PDI's initial release, implemented in `pdf_orchestrator.py`.
* **Advanced HyDE Question Generation:**
    * **PDI Implementation:** The enhanced `qa_generator.py` will be ported from the CFI to the PDI.
* **Knowledge Graph Triplet Extraction:**
    * **PDI Implementation:** The new module will be ported from the CFI to the PDI.
* **Multi-Page Question/Answer Synthesis:**
    * **PDI Implementation:** The core synthesis logic will be ported, but the trigger mechanism will be adapted to recognize and resolve PDF filename references found within the text.