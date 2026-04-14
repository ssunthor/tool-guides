# Legal Knowledge Base (LKB) - Design Document

**Version:** 0.3.0
**Date:** 2026-03-23
**Author:** Martin Kecka, Gemini AI
**Category:** Suite (I + P + R + A)
**Acronym:** LKB

---

## 1. Executive Summary

The **Legal Knowledge Base (LKB)** is a modular, AI-native contract intelligence platform designed to completely digitize, analyze, and query the company's legal agreements (MSAs, Amendments, SOWs). Positioned as a next-generation replacement for traditional Contract Lifecycle Management (CLM) systems like Ironclad, the LKB provides deep, semantic interrogation of unstructured historical data alongside structured metadata management.

To align with "Project CMDMeta" goals (automating the extraction of ~140 metadata properties) and ensure scalability for thousands of complex documents, the solution is decoupled into four independent applications:

1.  **Legal Document Parser (LDP - Processor):** A heavy-duty, asynchronous conversion engine that transforms raw files (PDF, Image, DOCX, XLSX) into standardized Markdown using a queue-based worker architecture.
2.  **Legal Knowledge Ingestor (LKI - Ingestor):** A semantic engine that processes Markdown into Vector Embeddings and executes batch LLM prompts to extract the "Static Answers" (Metadata) for the 140 required fields.
3.  **Legal Contract Retriever (LCR - Retriever):** A hybrid search engine exposed via gRPC that seamlessly routes queries to PostgreSQL (for exact metadata matching) or ChromaDB (for semantic clause retrieval).
4.  **Legal Knowledge Assistant (LKA - Assistant):** A user-facing web application (Split-Screen Dashboard) for browsing contracts, querying the database, verifying AI-extracted metadata, and handling "Human-in-the-Loop" (HITL) corrections and dynamic field generation.

---

## 2. Architecture & Reuse Strategy

This suite leverages the "Best of Breed" logic from existing SERA applications to minimize new code while establishing a highly specialized legal domain boundary.

### 2.1 Component Reuse Matrix

| Source Module | Component / Logic | Adaptation for Legal Suite |
| :--- | :--- | :--- |
| **CAP (CV Processor)** | **Async Worker Pattern** (`core/worker.py`) | **Core of LDP.** Reuse the Redis/Celery architecture to handle bulk processing of heavy PDF files without blocking the API. |
| **RMI (Remittance)** | **The Extraction Gauntlet** (`core/pipeline/a1_extraction`) | **Core of LDP.** Reuse the logic of trying sequential parsers (Layout -> OCR -> Native Text) until one succeeds. Added logic for `.docx` and `.xlsx` parsing. |
| **CFI (Confluence)** | **HyDE Generator** (`core/qa_generator.py`) | **Core of LKI.** Critical for legal RAG. Generates questions like *"What are the termination conditions?"* to match clauses about *"Material Breach"*. |
| **CFI (Confluence)** | **Map-Reduce Summarizer** (`core/llm_summarizer.py`) | **Core of LKI.** MSAs are token-heavy. We reuse the `_map_and_assemble_sections` logic for summaries. |
| **CFI (Confluence)** | **Semantic Analyzer** (`core/llm_analyzer.py`) | **Core of LKI.** Refactored into `MetadataExtractor`. Iterates through the **140 CMD Metadata fields** during ingestion to extract specific answers to PostgreSQL. |
| **CFR (Confluence Retriever)** | **Intent Routing & Reranking** | **Core of LCR.** Adapted for legal Hybrid Search (SQL vs. Vector) and NVIDIA NIM reranking. |
| **CAP (CV Processor)** | **UI / UX Pattern** | **Core of LKA.** Reuse the Dashboard layout, Split-Screen (Document vs. Data), and "Confidence Badge" logic for metadata verification. |
| **Shared Lib** | `LLMInterface`, `Embedder`, `VdbWriter` | Moved to `ai/common_lib` and imported by all components. |

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
    * *Logic:* Splits by Markdown Headers (`#`, `##`, `###`) which represent legal clauses (e.g., "1.1 Confidentiality").
3.  **`MetadataExtractionStep` (The "140 Fields" Logic)**:
    * *Input:* Loads the 140 metadata definitions (Yes/No, Dates, Entities, Clause Checks) from the Database.
    * *Process:* Utilizing the massive local H100 compute cluster, it brute-forces prompt batches across the chunks to pre-compute the answers for all 140 fields.
    * *Output:* Structured JSON stored in the `contract_metadata` PostgreSQL table.
4.  **`AdvancedEnrichmentStep`**:
    * **HyDE:** Creates hypothetical questions for complex clauses to aid vector retrieval.
    * **Graph:** Maps entity relationships.
5.  **`EmbedAndStoreStep`**:
    * Stores vectors in the ChromaDB `legal_contracts` collection.

---

## 5. Application 3: Legal Contract Retriever (LCR)

**Category:** (R) Retriever
**Role:** The "Hybrid Search Engine"
**Input:** Orchestrator Search Query (gRPC)
**Output:** Structured JSON containing SQL metadata and/or Vector citations.

The LCR is a dedicated gRPC service that acts as the bridge between the broader Orchestrator ecosystem and the Legal Knowledge Base. Because contract queries can be strictly factual ("What is the term?") or highly interpretive ("Summarize the liability limits"), the LCR uses a hybrid retrieval strategy.

### 5.1 Query Intent Routing
When passed a user query, a fast, small LLM (e.g., `gemma3:8b`) performs Intent Classification to route the search:

1.  **Metadata/SQL Intent:** If the query maps to one of the 140 pre-extracted fields (e.g., "Show me all contracts with Auto-Renew"), the LCR translates the query into a PostgreSQL `SELECT` statement against the `contract_metadata` table.
2.  **Semantic/Vector Intent:** If the query is open-ended or analytical (e.g., "How does the indemnification clause handle data breaches?"), the LCR converts the query to an embedding and searches ChromaDB.
3.  **Hybrid Intent:** If the query requires both (e.g., "Summarize the SLA penalties for contracts expiring in 2026"), the LCR filters by date in SQL, then performs a vector search only on the chunks belonging to that filtered subset of documents.

### 5.2 Context Formatting & Output
To prevent hallucinations in downstream synthesis, the LCR returns a strictly formatted payload:
* **SQL Hits:** Returned as clean key-value pairs representing the extracted fields.
* **Vector Hits:** Returned with the exact `clause_text`, the `document_name`, and a `citation_link` (e.g., `#page=4&chunk_id=102`) so the LKA UI can automatically scroll the user to the source material.
* **Reranking:** Vector results are passed through an NVIDIA NIM reranker to ensure only the highest-confidence clauses are returned.

---

## 6. Application 4: Legal Knowledge Assistant (LKA)

**Category:** (A) Assistant
**Role:** The "Interface" (Ironclad Replacement)
**Input:** User Query / UI Interaction
**Output:** RAG Response + Split-Screen Metadata Dashboard

### 6.1 Architecture
Built as a modern React SPA + FastAPI backend. It serves as the primary interactive portal for Legal, Deal Desk, and Finance teams.

### 6.2 UI Features
* **Split View:** The left pane renders the original Document (Markdown or PDF viewer). The right pane displays the Chat interface and the 140-field Metadata panel.
* **Validation Interface (HITL):** Users can view the AI-extracted fields.
    * *Confidence High:* Displayed normally.
    * *Confidence Low:* Displayed with a warning badge (needs review).
    * *Correction:* Users can click to edit the value. This "Human-in-the-Loop" feedback immediately updates the PostgreSQL database and flags the field as "Human Verified."
* **Citation Auto-Scroll:** Clicking an extracted field or an AI chat citation automatically navigates the left pane to the exact clause in the document.

---

## 7. Implementation Plan & Roadmap

Given that the core Parser (LDP) is largely complete, the remaining build is prioritized to deliver immediate, interactive value to the business.

### Phase 1: The "Read-Only" Dashboard (LKA - UI/Backend)
* Build the core Split-Screen React UI and FastAPI backend.
* Implement the backend endpoints to fetch the parsed Markdown and the 140 pre-extracted fields directly from PostgreSQL.
* **Goal:** Allow users to browse their contract repository and immediately see the AI's metadata extractions.

### Phase 2: Human-in-the-Loop (HITL) Integration
* Add edit capabilities to the 140 fields in the UI.
* Build the `PUT` endpoints in FastAPI to save corrections back to the database.
* Implement audit logging (tracking who changed what field and when).
* **Goal:** Establish trust by allowing legal professionals to verify and correct the AI's work.

### Phase 3: The Hybrid Retriever (LCR)
* Build the `gRPC` service that handles open-ended chat queries.
* Implement the SQL/Vector intent routing logic.
* Connect the LCR to the LKA chat interface.
* **Goal:** Enable complex, conversational Q&A against the contract repository with precise citations.

### Phase 4: Dynamic Field Generation
* Add a UI feature allowing authorized users to define a "141st custom field" (e.g., "Does this contract mention the new 2026 tax regulation?").
* Build the backend logic to spin up a Celery task that retroactively runs the new extraction prompt against the entire database of processed markdown chunks.
* **Goal:** Future-proof the platform by making the extraction engine entirely self-service for the business.