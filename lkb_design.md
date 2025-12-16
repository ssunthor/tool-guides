# Legal Knowledge Base (LKB) - Design Document

**Version:** 0.2.0
**Date:** 2025-12-01
**Author:** Martin Kecka, Gemini AI
**Category:** Suite (I + P + A)
**Acronym:** LKB

---

## 1. Executive Summary

The **Legal Knowledge Base (LKB)** is a modular AI ecosystem designed to digitize, analyze, and query the company's legal agreements (MSAs, Amendments, SOWs).

Unlike the Confluence Ingestor which handles structured web content, the LKB must process a mix of file formats—primarily complex PDFs (digital and scanned), but also Word documents (`.docx`) and Excel files used for non-standard Service Order Forms (SOFs).

To align with the "Project CMDMeta" goals (automating the extraction of ~140 metadata properties) and ensure scalability for thousands of historical documents, the solution is decoupled into three independent applications:

1.  **Legal Document Parser (LDP - Processor):** A heavy-duty, asynchronous conversion engine that transforms raw files (PDF, Image, DOCX, XLSX) into standardized Markdown using a queue-based worker architecture.
2.  **Legal Knowledge Ingestor (LKI - Ingestor):** A semantic engine that processes Markdown into Vector Embeddings and extracts the "Static Answers" (Metadata) for the 140 required fields.
3.  **Legal Knowledge Assistant (LKA - Assistant):** A user-facing application for querying contracts, verifying the AI-extracted metadata, and handling "Human-in-the-Loop" corrections.

---

## 2. Architecture & Reuse Strategy

This suite leverages the "Best of Breed" logic from existing SERA applications to minimize new code.



### 2.1 Component Reuse Matrix

| Source Module | Component / Logic | Adaptation for Legal Suite |
| :--- | :--- | :--- |
| **CAP (CV Processor)** | **Async Worker Pattern** (`core/worker.py`) | **Core of LDP.** Reuse the Redis/Celery architecture to handle bulk processing of heavy PDF files without blocking the API. |
| **RMI (Remittance)** | **The Extraction Gauntlet** (`core/pipeline/a1_extraction`) | **Core of LDP.** Reuse the logic of trying sequential parsers (Layout -> OCR -> Native Text) until one succeeds. Added logic for `.docx` and `.xlsx` parsing. |
| **CFI (Confluence)** | **HyDE Generator** (`core/qa_generator.py`) | **Core of LKI.** Critical for legal RAG. Generates questions like *"What are the termination conditions?"* to match clauses about *"Material Breach"*. |
| **CFI (Confluence)** | **Map-Reduce Summarizer** (`core/llm_summarizer.py`) | **Core of LKI.** MSAs are token-heavy. We reuse the `_map_and_assemble_sections` logic for summaries. |
| **CFI (Confluence)** | **Semantic Analyzer** (`core/llm_analyzer.py`) | **Core of LKI.** Refactored into `MetadataExtractor`. Instead of "User Intent", it iterates through the **140 CMD Metadata fields** to extract specific answers. |
| **CFI (Confluence)** | **Knowledge Graph** (`core/knowledge_graph_extractor.py`) | **Core of LKI.** Maps relationships (e.g., `GTT` -> `Indemnifies` -> `Client`). |
| **CAP (CV Processor)** | **UI / UX Pattern** | **Core of LKA.** Reuse the Dashboard layout, Split-Screen (Document vs. Data), and "Confidence Badge" logic for metadata verification. |
| **Shared Lib** | `LLMInterface`, `Embedder`, `VdbWriter` | Moved to `ai/common_lib` and imported by all three. |

---

## 3. Application 1: Legal Document Parser (LDP)

**Category:** (P) Processor
**Role:** The "Grinder"
**Input:** Raw Files (PDF, Images, DOCX, XLSX)
**Output:** Clean Markdown (`content.md`) + Layout JSON + Extracted Images

### 3.1 Asynchronous Architecture (Redis/Celery)
Processing a single scanned MSA can take minutes. To handle the requirement of "thousands of PDFs," LDP uses the **CAP** worker pattern.

* **API Service (FastAPI):** Accepts file uploads, validates format, saves to shared storage, and pushes a `process_document_task` to Redis. Returns a `job_id` immediately.
* **Redis:** Message Broker.
* **Worker Cluster (Celery):** Scalable workers pick up tasks. They run the heavy "Gauntlet" logic.
* **Result Backend:** Updates the job status in Postgres (`processing`, `completed`, `failed`).

### 3.2 Core Logic: The "Gauntlet" (Expanded from RMI)
Legal documents vary wildly. The workers execute an expanded "Gauntlet" pipeline:

1.  **Stage 1: File Type Routing**
    * **Office Files (.docx, .xlsx):** Route to native parsers (`python-docx`, `pandas`) to convert directly to Markdown/Tables.
    * **PDF/Images:** Proceed to Layout Analysis.
2.  **Stage 2: Digital Layout Analysis (`pymupdf` / `surya`)**
    * Checks if the PDF has a clean, selectable text layer.
    * Extracts text while preserving visual hierarchy (headers vs. body) using bounding box analysis.
3.  **Stage 3: Intelligent OCR (`marker` / `tesseract`)**
    * If Stage 2 fails (scanned image), attempts local OCR optimized for document layout (detecting columns).
4.  **Stage 4: Vision Fallback (`nanonets` / `gemma3:27b`)**
    * For extremely noisy scans or handwritten notes.

### 3.3 Deliverable
The LDP outputs a folder per document containing:
* **`content.md`**: The contract text in strict Markdown. Headers (`# 1. Definitions`) are preserved.
* **`images/`**: Extracted figures/tables.
* **`metadata.json`**: Technical metadata (page count, OCR confidence score, parser used).

---

## 4. Application 2: Legal Knowledge Ingestor (LKI)

**Category:** (I) Ingestor
**Role:** The "Brain"
**Input:** Markdown (from LDP output)
**Output:** Postgres (Structured Metadata), ChromaDB (Vectors)

This is a **Dual-Mode** application (Incremental Sync & Standalone Task). It picks up completed jobs from the LDP output folder.

### 4.1 Workflow Steps

1.  **`FetchProcessedDocStep`**: Loads the Markdown from the LDP output directory.
2.  **`SemanticChunkingStep`**:
    * *Reuse:* `text_chunker.py` (CFI).
    * *Logic:* Splits by Markdown Headers (`#`, `##`, `###`) which represent legal clauses (e.g., "1.1 Confidentiality").
3.  **`MetadataExtractionStep` (The "140 Fields" Logic)**:
    * *Reuse:* `llm_analyzer.py` (CFI).
    * *Input:* Loads the 140 metadata definitions (Yes/No, Dates, Entities) from the Database.
    * *Process:* For every required field (e.g., "Effective Date", "Change of Control Clause?"), it prompts the LLM with relevant text chunks.
    * *Output:* Structured JSON stored in the `contract_metadata` table.
4.  **`AdvancedEnrichmentStep`**:
    * **HyDE:** Runs `qa_generator.py` (CFI) to create hypothetical questions for complex clauses.
    * **Graph:** Runs `knowledge_graph_extractor.py` (CFI) to map entity relationships.
5.  **`EmbedAndStoreStep`**:
    * *Reuse:* `embedder.py` (CFI).
    * Stores vectors in ChromaDB `legal_contracts` collection.

---

## 5. Application 3: Legal Knowledge Assistant (LKA)

**Category:** (A) Assistant
**Role:** The "Interface"
**Input:** User Query / UI Interaction
**Output:** RAG Response + Metadata Dashboard

### 5.1 Architecture
Built as a React SPA + FastAPI backend, mirroring the **CAP** architecture.

### 5.2 The "Hybrid Search" Engine
This handles the requirement to query specific metadata vs. general concepts.

* **Scenario A: "Metadata Query"**
    * *User:* "Which contracts have a Change of Control clause?"
    * *Logic:* The Agent queries the SQL `contract_metadata` table directly.
    * *Result:* Returns a precise list of contracts. 100% accurate, no hallucination.
* **Scenario B: "Conceptual Query"**
    * *User:* "Summarize the indemnification obligations for GTT in the Insulet MSA."
    * *Logic:* Triggers the **Advanced RAG Pipeline** (using HyDE and Reranking) to find relevant clauses and synthesize an answer.

### 5.3 UI Features (The CAP Influence)
* **Split View:** Left side shows the Original Document (scrolled to the relevant clause). Right side shows the Chat/Metadata panel.
* **Validation Interface:** Users (e.g., Sales Admin/Legal) can view the AI-extracted "140 Fields".
    * *Confidence High:* Displayed in Green.
    * *Confidence Low:* Displayed in Red (needs review).
    * *Correction:* Users can edit the value. This "Human-in-the-Loop" feedback updates the DB and helps fine-tune the system.

---

## 6. Implementation Plan: Phase 1 (The Parser)

Our immediate focus is the **Legal Document Parser (LDP)**.

### 6.1 Requirements
* **Stack:** Python, PyMuPDF (fitz), Marker (surya-ocr), LayoutParser, python-docx, pandas (for Excel), **Celery, Redis**.
* **Dependencies:** `torch`, `transformers` (for Marker).
* **Output:** Strict Markdown.

### 6.2 Task List
1.  **Scaffold Repository:** Create `ai/legal_document_parser`.
2.  **Infrastructure Setup:** Configure the Redis container and Celery worker entrypoint in `docker-compose`.
3.  **Ingest Test Files:** Create a test suite including Clean/Dirty PDFs and Non-standard SOFs.
4.  **Benchmark:** Run `marker` (local) vs. `nanonets` (API) on the test set.
5.  **Develop `layout_to_markdown.py`**: A robust script to convert the layout analysis into Markdown that preserves legal numbering (1.1, 1.1.a) and table structures.