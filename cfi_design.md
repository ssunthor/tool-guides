# /ai/assets/docs/tool_guides/cfi_design.md
# Confluence Ingestor (CFI) - Design Document

**Version:** 1.3
**Date:** 2025-09-02  
**Author:** Martin Kecka  
**Category:** Ingestor (I)  
**Acronym:** CFI (Confluence Ingestor)  

---

## 1. Overview

The Confluence Ingestor is a Python application designed to connect to a Confluence instance, extract content (pages, images, attachments) from specified spaces, process this content, enrich it with AI-generated metadata, and ingest it into the AI Assistant's data stores. These stores include:

* **PostgreSQL:** For detailed metadata about pages, chunks, assets, extracted keywords, LLM-generated summaries, semantic analysis, category hierarchy, synthesized Q&A pairs, and links between related data.
* **Vector Database (ChromaDB):**
    * **Chunk Collection:** For text chunks derived from page content and their vector embeddings.
    * **Page Summary Collection:** For page-level summary embeddings to facilitate two-stage retrieval strategies.
    * **Question Collection:** For embeddings of hypothetical questions (HyDE) to improve search relevance.
* **Shared Asset Storage:** For raw downloaded files (images, PDFs, DOCX, etc.) accessible via a shared Persistent Volume Claim (PVC) in Kubernetes environments or a local directory during development.

This ingestor is a modular component crucial for populating the knowledge base used by the AI Assistant.

---

## 2. Features

* Connects to Confluence Cloud or Server (configurable).
* Ingests content from a configurable list of Space Keys or a specific Page ID.
* Supports full sync and incremental sync (based on content hash and last updated timestamp).
* **Dual-Mode Architecture:**
    * **Incremental Sync (Default):** A multi-step pipeline that processes pages one by one, with change detection to efficiently skip unchanged content.
    * **Standalone Task Mode:** Allows for bulk execution of specific enrichment tasks (e.g., generating all summaries, embeddings) across the entire database, using granular timestamps to process only the necessary items.
* Extracts main page content (HTML) and attachment details.
* Parses HTML to extract clean text, identify images, attachments, and internal links, handling various Confluence macros.
* Downloads images and specified attachments, storing them in a structured manner.
* Reconstructs and stores the Confluence category hierarchy for each processed space in PostgreSQL.
* Chunks extracted text content using configurable strategies (e.g., recursive character splitting).
* Generates embeddings for text chunks and page summaries using configurable models (Ollama, NVIDIA NIM).
* **AI Enrichment (Optional & Bulk-Processable):**
    * Generates page and chunk summaries (both detailed first-pass and concise second-pass) using an LLM.
    * Generates Hypothetical Document Embeddings (HyDE) by creating relevant, intent-driven questions for content to enhance search and RAG capabilities.
    * Identifies and summarizes complex tables within content chunks using an LLM.
    * Performs semantic analysis on chunks to generate structured metadata (e.g., document type, user intent, key entities) using an LLM.
    * Processes images using a vision model to extract keywords and descriptions.
    * Synthesizes complex Q&A pairs that require combining information from multiple, linked documents.
* Manages data consistency across PostgreSQL, Vector DB, and Shared Asset Storage.
* Comprehensive logging and context-aware statistics reporting for observability.
* Includes a bootstrap helper for virtual environment and dependency management.

---

## 3. Architecture & Core Components

The application is structured into a main dispatcher, a pipeline orchestrator for incremental syncs, and a set of core modules responsible for specific logic.

* `confluence_ingestor.py`: The main entry point and task dispatcher. It parses CLI arguments and decides whether to run the default incremental sync or a specific standalone task.
* `core/processor.py` (**ConfluenceProcessor**): The central orchestrator for the incremental sync pipeline. It executes a sequence of modular workflow steps for each page.
* `core/workflow/`: This directory contains the modular, single-responsibility steps that form the incremental processing pipeline (e.g., `step_01_fetch_data.py`, `step_06_process_html_and_chunk.py`, etc.). These steps act as thin wrappers around the core logic modules.
* **Core Logic Modules (`core/`):** These modules contain the primary business logic and are designed for dual-mode operation (supporting both incremental and standalone execution).
    * `core/confluence_client.py`: Manages all communication with the Confluence API.
    * `core/html_processor.py`: Parses Confluence HTML, cleans it, and extracts text, tables, links, and other structured data.
    * `core/text_chunker.py`: Splits large text content into smaller, manageable chunks.
    * `core/embedder.py`: Generates vector embeddings using configured models.
    * `core/image_processor.py`: Processes images using a vision model.
    * `core/llm_summarizer.py`: Generates summaries for pages and chunks.
    * `core/qa_generator.py`: Generates hypothetical questions (HyDE).
    * `core/table_summarizer.py`: Generates natural language summaries of tables.
    * `core/llm_analyzer.py`: Generates structured semantic metadata for chunks.
    * `core/multi_page_qa_synthesizer.py`: Generates Q&A pairs synthesized from multiple documents.
    * `core/llm_interface.py`: A centralized interface for making calls to LLM services.
* **Data Handling & Storage:**
    * `core/sql_writer.py`: Handles all interactions with the PostgreSQL database.
    * `core/vdb_writer.py`: Manages writing data to the Vector Database (ChromaDB).
    * `core/asset_writer.py`: Handles saving downloaded assets to the shared storage.
    * `database/models.py`: Defines the SQLAlchemy ORM models for all PostgreSQL tables.
* **Configuration & Utilities:**
    * `core/config_loader.py`: Loads and merges configuration from `settings.yaml` and environment variables.
    * `prompts/`: A centralized directory containing Python files with all LLM prompt templates.
    * `core/statistics.py`: Manages tracking and reporting of metrics for each run.
    * `bootstrap_helper.py`: A utility script to manage the Python virtual environment and dependencies.
    * `utils/logger.py`: Manages logging behavior across the application.

---

## 4. Process Flow

The ingestor operates in one of two primary modes, determined by CLI arguments.

### 4.1. Mode 1: Incremental Sync (Default)

This mode is orchestrated by the `ConfluenceProcessor` and executes a linear pipeline of steps for each page in a space.

1.  **Initialization:** The processor loads the configuration and initializes all workflow step modules.
2.  **Target Identification:** It fetches the list of pages for each configured `space_key`.
3.  **Page Processing Loop:** For each page, a `PageProcessingContext` object is created and passed through the following steps:
    1.  **FetchDataStep:** Retrieves page HTML, metadata, and attachments.
    2.  **ChangeDetectionStep:** Compares the page's content hash/timestamp to the database to determine if reprocessing is needed. If not, the pipeline for this page is halted.
    3.  **ProcessAttachmentsStep:** Downloads and saves attachments.
    4.  **ProcessVisionStep:** (If enabled) Processes images using a vision model.
    5.  **CleanupReprocessedPageStep:** If the page has changed, this step deletes all of its old data (chunks, summaries, embeddings) from PostgreSQL and the Vector DB to prevent stale information.
    6.  **ProcessHtmlAndChunkStep:** Parses the HTML, extracts text, tables, links, and creates initial text chunks.
    7.  **GenerateTableSummariesStep:** (If enabled) Generates natural language summaries for tables within chunks.
    8.  **GenerateSummariesStep:** (If enabled) Generates detailed and brief summaries for the page and each chunk.
    9.  **RunSemanticAnalysisStep:** (If enabled) Generates structured semantic metadata for each chunk.
    10. **GenerateHydeQuestionsStep:** (If enabled) Generates hypothetical questions.
    11. **EnrichAndEmbedChunksStep:** Prepares the final text for embedding (using summaries, context, etc.) and generates the vector embeddings.
    12. **StoreDataStep:** Writes all the newly generated data (page metadata, chunks, summaries, questions, embeddings) to PostgreSQL and the Vector DB.
4.  **Post-Space Processing:** After all pages in a space are processed:
    1.  **CheckDeletionsStep:** Identifies and removes pages from the data stores that no longer exist in Confluence.
    2.  **BuildCategoriesStep:** Reconstructs the complete category hierarchy for the space.
    3.  **GenerateSynthesisStep:** Runs targeted multi-page Q&A synthesis on pages that were updated.

### 4.2. Mode 2: Standalone Task Mode

This mode is triggered by the `--run-task` CLI argument and is dispatched by `confluence_ingestor.py`.

1.  **Task Selection:** The user specifies a task to run (e.g., `summaries`, `hyde`, `embedding`).
2.  **Database Query:** The corresponding core module (e.g., `LLMSummarizer`) directly queries the PostgreSQL database. It uses the granular `last_*_at` timestamps to find all items that need processing.
3.  **Bulk Processing:** The module processes the identified items in batches, generating the required enrichments and updating the results in the database.
4.  **Timestamp Update:** Upon successful processing of an item, the corresponding timestamp (e.g., `last_embedded_at`) is updated.

---

## 5. Standalone Enrichment Tasks

These are post-processing or enrichment steps that can be run in bulk across the entire dataset using the Standalone Task Mode.

### Hypothetical Question Generation (HyDE)

* **Purpose:** To improve search relevance by generating and embedding hypothetical questions a user might ask.
* **Trigger:** `--run-task hyde`

### Table Summarization

* **Purpose:** To make tabular data accessible by generating natural-language summaries of tables.
* **Trigger:** `--run-task tables`

### LLM Semantic Analysis

* **Purpose:** To enrich text chunks with structured metadata (e.g., `document_type`, `user_intent`) for advanced filtering and routing.
* **Trigger:** `--run-task analysis`

### Vision Analysis

* **Purpose:** To generate keywords and descriptions for all images in the asset store.
* **Trigger:** `--run-task vision`

### Embedding Generation

* **Purpose:** To generate or regenerate vector embeddings for all chunks that need it.
* **Trigger:** `--run-task embedding`

### Multi-Page Q&A Synthesis

* **Purpose:** To create complex Q&A pairs that require synthesizing information from multiple, linked documents.
* **Trigger:** `--run-task synthesis`

---

## 6. Configuration

Configuration is managed through `config/settings.yaml` and can be overridden by environment variables.

### 6.1. `settings.yaml`

This YAML file contains ingestor-specific settings. Key sections include:

* **`confluence`:**
    * `url`, `username`, `api_token`: (Required) Credentials for the Confluence instance.
    * `space_keys`: (Required) A list of Confluence space keys to ingest.
    * `api_batch_size`: Number of items to fetch per API call.
* **`processing`:**
    * `chunking`: Defines the text splitting strategy (`chunk_size`, `chunk_overlap`).
    * `embedding`: (Required) Configuration for the embedding client (`nim` or `ollama`).
    * `llm_summarizer`, `qa_generation`, `table_summarization`, `llm_analyzer`: Sections to configure the respective AI enrichment tasks.
    * `attachments`: Master switch to download attachments, with `allowed_extensions` and `max_size_mb`.
    * `temp_download_dir`: (Required) A temporary local path for downloading assets.
* **`storage`:**
    * `asset_base_path`: (Required) The root path for the Shared Asset Storage.
    * `vdb`: (Required) Configuration for the vector database (`host`, `port`, `collection names`).
    * `postgresql`: (Required) The `dsn` connection string for the PostgreSQL database.
* **`logging`:**
    * `level`: Console logging level (e.g., `INFO`, `DEBUG`).
    * `local_dev_log_directory`: Path to store log files when running locally.

### 6.2. Environment Variables

Sensitive or deployment-specific settings can be overridden by environment variables. Key variables include:

* `CONFLUENCE_URL`, `CONFLUENCE_USERNAME`, `CONFLUENCE_API_TOKEN`
* `POSTGRES_DSN`
* `VDB_HOST`, `VDB_PORT`
* `LLM_OLLAMA_BASE_URL`
* Client-specific settings like `EMBEDDING_NIM_BASE_URL`, `LLM_SUMMARIZER_NIM_MODEL`, etc.
* API keys for services like NVIDIA NIM.

Refer to `core/config_loader.py` for the complete list.

---

## 7. Data Storage

### 7.1. PostgreSQL (`database/models.py`)

The PostgreSQL database is the source of truth for all structured metadata. Main tables include:

* **`sources_metadata`:** One record per Confluence page, storing high-level information like title, url, `space_key`, author, version, `content_hash`, and timestamps.
* **`source_chunks`:** Records for each text chunk, linked to a source page. This table is central to the dual-mode architecture and contains:
    * `raw_text_content`: The original text content of the chunk.
    * `structured_tables_json`, `image_details_json`, `internal_links_json`: Structured data extracted from the chunk.
    * `llm_analysis`: A JSON column for structured semantic metadata.
    * **Granular Timestamps:** A set of columns (`processed_at`, `last_summarized_at`, `last_hyde_generated_at`, `last_table_summarized_at`, `last_analyzed_at`, `last_embedded_at`) to track the status of each enrichment task.
* **`image_vision_metadata`:** Stores keywords and descriptions for each processed image.
* **`llm_summaries`:** Stores all LLM-generated summaries, typed by `page`, `chunk`, or `table`.
* **`generated_questions`:** Stores the hypothetical questions generated for HyDE.
* **`synthesized_qna`:** Stores complex questions and synthesized answers that require information from multiple source documents.
* **`confluence_category_hierarchy`:** Stores the reconstructed category tree for each space.
* **`failed_ingestion_items`:** Logs items that failed during processing for review.

### 7.2. Vector Database (ChromaDB)

* **Chunk Collection:** Stores embeddings of contextually enriched text chunks. Metadata includes `source_id`, `chunk_sequence_number`, `space_key`, etc., to link back to PostgreSQL.
* **Page Summary Collection:** Stores embeddings of brief page summaries for two-stage retrieval.
* **Question Collection:** Stores embeddings of the hypothetical questions generated by the HyDE process.

### 7.3. Shared Asset Storage

A filesystem (e.g., a Kubernetes Persistent Volume Claim) for storing raw binary files. Assets are saved in a structured path: `/ai/assets/images/confluence/<space_key>/<page_id>/<filename>`.

---

## 8. Command-Line Interface (CLI) Usage

The ingestor is run using `python confluence_ingestor.py [OPTIONS]`.

### Incremental Sync Mode (Default)

* `--settings FILE_PATH`: Path to the `settings.yaml` file.
* `--log-dir DIR_PATH`: Directory to store log files.
* `--log-level LEVEL`: Console logging level (DEBUG, INFO, etc.).
* `--space-key KEY`: Process only this specific space key.
* `--page-id ID`: Process only this specific page ID.
* `--force-reprocess`: Force reprocessing of all pages, ignoring change detection.

[bash]
# Run the standard incremental sync for the "SE" space.
python confluence_ingestor.py --space-key SE
[/bash]

### Standalone Task Mode

* `--run-task [summaries|hyde|tables|analysis|vision|embedding|synthesis|post-run|verify]`: Run a specific bulk processing task.
* `--force-regenerate`: Force regeneration for all items in the task.
* `--limit INT`: Limit the number of items processed by the task.
* `--cascade`: Automatically run downstream dependent tasks.

[bash]
# Run bulk HyDE question generation for all relevant content.
python confluence_ingestor.py --run-task hyde
[/bash]

---

## 9. Logging

* **Console Logging:** Outputs to `stderr`. Level is configurable via `--log-level` or `settings.yaml`.
* **File Logging:** Timestamped log files are created in the configured log directory. File logs are always at `DEBUG` level for maximum detail. Log rotation is handled by keeping a configured number of recent logs.
* **Format:** `YYYY-MM-DD HH:MM:SS [%(levelname)-5s] %(name)s (%(module)s.%(funcName)s): %(message)s`

---

## 10. Key Dependencies

* `atlassian-python-api`: For Confluence API interaction.
* `beautifulsoup4`, `lxml`: For HTML parsing.
* `SQLAlchemy`, `psycopg2-binary`: For PostgreSQL interaction.
* `chromadb`: For Vector DB interaction.
* `langchain`: For text splitting utilities.
* `requests`: For HTTP requests to LLM/NIM APIs.
* `python-dotenv`: For loading `.env` files.
* `PyYAML`: For loading YAML configuration.

Refer to `requirements.txt` for a complete list.

---

## 11. Future Enhancements

The following enhancements are planned to be built on top of the current dual-mode architecture.

### Knowledge Graph Triplet Extraction (In Progress)

* **Objective:** To structure the unstructured text within Confluence into a formal knowledge graph, enabling precise, multi-hop queries.
* **User Value 🌟:** Transforms search from keyword matching to a structured query engine. Users can ask specific, relational questions like, "Show me all services that use the 'Prometheus' tool and are managed by the 'Platform Team'."
* **Implementation Strategy 🛠️**
    * **Status:** The core module (`knowledge_graph_extractor.py`), database model (`KnowledgeTriplet`), and CLI trigger (`--run-task triplets`) are implemented.
    * **Next Steps:** The feature requires full integration into the post-run workflow, rigorous testing, and potential performance optimization before it can be considered production-ready. A new `knowledge_triplets` table will store the extracted (Subject, Predicate, Object) data.

### Advanced Statistics and Observability

* **Objective:** To transform the statistics module from a simple counter into a comprehensive observability system. This will provide clear, context-aware reports for both incremental syncs and standalone tasks.
* **User Value 🌟:** Operators will be able to instantly understand the impact of an ingestion run (e.g., "2 new pages, 5 updated pages") and diagnose issues (e.g., "20 unresolved links pending category build") without manually parsing debug logs.
* **Implementation Strategy 🛠️**
    * **Refactor `IngestionStatistics`:** Enhance the class to track more granular, context-aware metrics (e.g., `pages_new`, `pages_updated`, `links_unresolved`).
    * **Integrate Tracking into Workflow:** Update relevant steps (e.g., `ChangeDetectionStep`) to increment the new, specific counters.
    * **Enhance Standalone Tasks:** Modify `run_as_standalone_task` methods to return rich dictionaries of their activities (e.g., `{ "pages_processed": 5, "chunk_summaries_generated": 30 }`).
    * **Implement Unified Reporting:** Create a single `report_summary(run_mode: str)` method that generates a "Delta Summary" for incremental mode and a task-specific summary for standalone mode.

### Configuration & Code Refinements (DONE)

* **Objective:** To improve maintainability and robustness by centralizing shared configuration values and refactoring minor code duplications.
* **User Value 🌟:** Reduces the risk of configuration drift between modules and simplifies future development by ensuring a single source of truth for operational parameters.
* **Implementation Strategy 🛠️**
    * Consolidate hardcoded values (e.g., minimum word counts for processing currently in `qa_generator.py` and `sql_writer.py`) into `settings.yaml`.
    * Conduct a full review of `settings.yaml` to ensure clarity, consistency, and completeness for all configurable parameters.
    * Move image-related LLM prompts into their own dedicated files within the `prompts/` directory.

### Refactor into a Universal Ingestor Framework (LATER STAGE)

* **Objective:** To re-architect the Confluence-specific ingestor into a modular, two-layer **Universal Ingestor Framework**. This framework will consist of source-specific **"Fetcher"** plugins and reusable file-type **"Parser"** plugins, feeding a common processing pipeline.
* **User Value 🌟:** This is a strategic evolution that unlocks the ability to ingest knowledge from any enterprise source (SharePoint, network drives, etc.), not just Confluence. It transforms the ingestor from a single-purpose tool into the central knowledge pipeline for the entire SERA ecosystem.
* **Implementation Strategy 🛠️**
    1.  **Core Refactor:** Abstract the current processing logic into a generic pipeline.
    2.  **Create "Parser" Plugins:** Develop a set of reusable parsers for key file types (e.g., PDF, DOCX, PPTX). This is a critical first step, as it enables the processing of attachments from *any* source. This directly addresses the dependency for CFR to query attachment content.
    3.  **Create "Fetcher" Plugins:** Develop source-specific plugins that know how to connect to a source, fetch files, and hand them off to the appropriate parser.
        * The existing Confluence logic will be refactored into the first `ConfluenceFetcher`.
        * New fetchers will be created for sources like `SharePoint` and `DFS` network shares.

### BUG FIX of status label ingesting (FIXED)

* **CRITICAL** CFI is not properly extracting status labels from confluence just simple leaving them empty... this is mainly problematic for sera to do list in confluence where then end result is half empty table. This need to be priority to fix. Testing page-id = 1559756825

### Pre-computed Intent-Based Summaries (work in progress) -> included also undocumented revamp of generic llm summaries to 4 pass logic -> pending RAG part

* **Objective:** To dramatically improve query-time performance and relevance for common user intents by pre-computing multiple, tailored summaries during the ingestion phase. This strategy shifts complexity from the real-time retrieval (CFR) to the offline ingestion (CFI) process.
* **User Value 🌟:** Blazing-fast retrieval for common questions, as the system can perform a direct lookup instead of a complex RAG process. Provides higher quality, more relevant context to the final reasoning LLM, leading to better answers. Reduces real-time computational load on the AI Factory.
* **Implementation Strategy 🛠️**
    1.  **Define Personas/Intents:** A set of standard user intents (e.g., `executive_summary`, `technical_deep_dive`, `troubleshooting_guide`, `sales_pitch`,) will be defined in a configuration file.
    2.  **Create Tailored Prompts:** For each intent, a specific, high-quality summarization prompt will be created in the `prompts/` directory.
    3.  **Enhance Ingestor Workflow:** The CFI's `GenerateSummariesStep` will be modified to iterate through the defined intents, calling the `LLMSummarizer` for each document with the corresponding tailored prompt.
    4.  **Update Database Schema:** The `llm_summaries` table in PostgreSQL will be updated to include an `intent_type` column to tag each summary.
    5.  **Update Retriever (CFR) Logic:** The CFR will be enhanced so that its first step is to classify the user's query intent. For matching intents, it will then perform a direct lookup in PostgreSQL to fetch the specific, pre-computed summary for the relevant document, bypassing more complex RAG strategies for common query types.

### Pre-computed Procedural Workflow Summaries (to be re-evaluated based on triplet implementation)

* **Objective:** To solve the challenge of answering broad, multi-document procedural queries (e.g., "How do I provision a VM?") by pre-computing a single, cohesive summary during ingestion that describes a document's role within the end-to-end process.
* **User Value 🌟:** Enables the AI to provide a complete, step-by-step overview of a complex process in a single, coherent response, rather than just summarizing one piece of the puzzle. This dramatically improves the user experience for procedural and "how-to" questions.
* **Implementation Strategy 🛠️**
    1.  **Define New "Procedural Workflow" Intent:** Create a new intent type for summaries that is specifically designed to capture a document's place in a larger workflow.
    2.  **Create Specialized Prompt:** Develop a new, dedicated summarization prompt. This prompt will instruct the LLM to analyze a page's content and its links to other known process documents to synthesize a "Process Context Summary." The summary will explicitly outline the "Overall Process," "Prerequisites," "Key Actions on This Page," and "Next Steps."
    3.  **Enhance Ingestor Workflow:** The CFI will be updated to identify documents that are part of a known procedural workflow (based on their links and content). For these documents, it will trigger this new summarization prompt in addition to the standard ones.
    4.  **Update Database Schema:** The `llm_summaries` table will be updated to store this new `procedural_workflow` intent type.
    5.  **Update Retriever (CFR) Logic:** The CFR will be enhanced to detect procedural queries. When detected, it will be programmed to specifically fetch this pre-computed workflow summary, providing the final reasoning LLM with a perfect, end-to-end overview of the process.

### Summaries (draft):
New logic of four phase summaries should be revisited to be applied based on document type... current logic using 20k characters is not enough... For Pages like Netbox Provisioning which are 8k characters it is correctly triggered simple 2 pass summary which is superior, but the logic should be adapted and tested when is more handy to use the 4 pass summaries and when just 2 pass based on document type...

Conclusion & Recommendation
This is a crucial learning moment for the summarization system. It demonstrates that the Four-Phase narrative synthesis, while powerful, is not a universal solution. The system needs a way to recognize when a document is a procedural guide or a checklist and adapt its strategy accordingly.

For this specific document, the Legacy summary should be used. The risk of missing a required field during provisioning is too high with the new version.

The ultimate goal for your system would be to build a "meta-model" or a preliminary classification step that decides which summarization pipeline to use based on the document's content:

If Document Type == "Conceptual/Architectural" -> Use the Four-Phase Narrative Pipeline.

If Document Type == "Procedural/Checklist/Guide" -> Use a structured extraction model similar to the one that produced the Legacy summary.

### Embedding enrichment upgrade:
Add purpose_summary from llmanalyze to enrichment embedding for documents.
Recommended Implementation
The cost of adding one more sentence is minimal, but the potential reward in search relevance is very high. I would recommend concatenating it with your existing enrichment sources.

To make the signal even stronger, you can add a prefix to clearly label its intent for the embedding model.

Your final text for embedding could look like this:

Adding IPs to Interfaces (Optional)

If it is required to pre-assign IP we can manually assign them, otherwise we can leave IP assigned to [Confluence Ref]...

Detailed Summary: This segment of the “Netbox Provisioning” guide... and setting the initial ‘Reserved’ status.

Table Summary: This table details the necessary IP information to be added... and it must match the prefix VRF.

PURPOSE: To explain how to optionally pre-assign IP addresses to interfaces in Netbox during provisioning, or allow CloudStack to automatically assign them.

The capitalized PURPOSE: prefix helps the model treat that final sentence as a distinct and important piece of metadata, further enhancing its impact on the final vector.