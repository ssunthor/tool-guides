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
2.  **Legal Document Ingestor (LDI - Ingestor):** A semantic engine that processes Markdown into Vector Embeddings and executes batch LLM prompts to extract the "Static Answers" (Metadata) for the 140 required fields.
3.  **Legal Document Retriever (LDR - Retriever):** A hybrid search engine exposed via gRPC that seamlessly routes queries to PostgreSQL (for exact metadata matching) or ChromaDB (for semantic clause retrieval).
4.  **Legal Document Assistant (LDA - Assistant):** A user-facing web application (Split-Screen Dashboard) for browsing contracts, querying the database, verifying AI-extracted metadata, and handling "Human-in-the-Loop" (HITL) corrections and dynamic field generation.

---

## 2. Architecture & Reuse Strategy

This suite leverages the "Best of Breed" logic from existing SERA applications to minimize new code while establishing a highly specialized legal domain boundary.

### 2.1 Component Reuse Matrix

| Source Module | Component / Logic | Adaptation for Legal Suite |
| :--- | :--- | :--- |
| **CAP (CV Processor)** | **Async Worker Pattern** (`core/worker.py`) | **Core of LDP.** Reuse the Redis/Celery architecture to handle bulk processing of heavy PDF files without blocking the API. |
| **RMI (Remittance)** | **The Extraction Gauntlet** (`core/pipeline/a1_extraction`) | **Core of LDP.** Reuse the logic of trying sequential parsers (Layout -> OCR -> Native Text) until one succeeds. Added logic for `.docx` and `.xlsx` parsing. |
| **CFI (Confluence)** | **HyDE Generator** (`core/qa_generator.py`) | **Core of LDI.** Critical for legal RAG. Generates questions like *"What are the termination conditions?"* to match clauses about *"Material Breach"*. |
| **CFI (Confluence)** | **Map-Reduce Summarizer** (`core/llm_summarizer.py`) | **Core of LDI.** MSAs are token-heavy. We reuse the `_map_and_assemble_sections` logic for summaries. |
| **CFI (Confluence)** | **Semantic Analyzer** (`core/llm_analyzer.py`) | **Core of LDI.** Refactored into `MetadataExtractor`. Iterates through the **140 CMD Metadata fields** during ingestion to extract specific answers to PostgreSQL. |
| **CFR (Confluence Retriever)** | **Intent Routing & Reranking** | **Core of LDR.** Adapted for legal Hybrid Search (SQL vs. Vector) and NVIDIA NIM reranking. |
| **CAP (CV Processor)** | **UI / UX Pattern** | **Core of LDA.** Reuse the Dashboard layout, Split-Screen (Document vs. Data), and "Confidence Badge" logic for metadata verification. |
| **Shared Lib** | `LLMInterface`, `Embedder`, `VdbWriter` | Moved to `ai/common_lib` and imported by all components. |

---

## 3. Application 1: Legal Document Parser (LDP)

**Category:** (P) Processor
**Role:** The "Grinder"
**Input:** Raw Files (PDF, Images, DOCX, XLSX)
**Output:** Clean Markdown (`content_nanonets.md`), JSON Stats, and Repaired Tables

### 3.1 Asynchronous Map-Reduce Architecture
Processing a single scanned MSA can take minutes. To handle the requirement of "thousands of PDFs," the LDP uses a highly parallelized **Celery and Redis-backed Map-Reduce (Chords)** architecture.

* **API Service / Orchestrator:** Accepts file uploads, saves to shared storage, and pushes a master job to Postgres (`pending`).
* **Dispatcher (Celery Beat):** Polls for pending jobs and dispatches them based on cluster capacity to prevent GPU starvation.
* **Scatter (Page-Level OCR):** The master task rips the PDF into high-res images and dispatches a concurrent sub-task for *every single page*.
* **Gather (Stitching & Repair):** A callback chord stitches the processed pages back together and initiates the holistic repair sequence.

### 3.2 Core Logic: The Vision-First Pipeline
We have transitioned from legacy layout models to a purely vision-driven pipeline to handle the extreme variance in legal scans.

1.  **Stage 1: Concurrent Vision OCR (`nanonets`)**
    * Every page is sent concurrently to the Nanonets OCR2-3B endpoint, which reads the page naturally, capturing text and converting tables to HTML `<tr>`/`<td>` tags.
2.  **Stage 2: Holistic Table Repair (The "Double-Rip" Strategy)**
    * The system scans the stitched Markdown for broken HTML tags spanning page boundaries.
    * If a broken table is detected, it crops the bottom 50% of the originating page and the top 50% of the subsequent page.
    * It prompts **Gemma 4 (31B)** to reconstruct just the missing `<tr>` rows, seamlessly stitching them back into the document (with a Nanonets fallback if Gemma times out).
3.  **Stage 3: Intelligent Metadata Extraction (`gemma-3-27b-it`)**
    * **The Sandwich Method:** For documents exceeding 50k characters, the middle is gutted, leaving the top 75% and bottom 25% to fit perfectly within context windows.
    * **Logprob Confidence Scoring:** Extracts the base metadata (Category, Date, Customer) and calculates the exact token log-probabilities to generate a mathematical confidence score, rather than relying on LLM self-evaluations.
4.  **Stage 4: Entity Resolution (`cmc_client`)**
    * The extracted customer name/email is fired against the master CMD registry via gRPC to resolve the official Client ID and Group ID.

---

## 4. Application 2: Legal Document Ingestor (LDI)

**Category:** (I) Ingestor
**Role:** The "Brain"
**Input:** Markdown (from LDP output)
**Output:** Postgres (Structured Metadata), ChromaDB (Vectors)

This is a **Dual-Mode** application (Incremental Sync & Standalone Task). It picks up completed jobs from the LDP output folder.

### 4.1 Workflow Steps

1.  **`FetchProcessedDocStep`**: Loads the Markdown from the LDP output directory.
2.  **`SemanticChunkingStep` (Hybrid LLM + Deterministic Strategy)**:
    * *Logic:* Instead of relying on fragile Markdown headers, the LDI assigns line numbers to the text and feeds it to an LLM (Gemma 4) using sliding overlapping windows. The LLM extracts semantic pointers (start/end lines, chunk types, TOC numbers). A deterministic Python structural merger then deduplicates the overlaps, stitches broken boundaries, and isolates tables.
3.  **`MetadataExtractionStep` (The "140 Fields" Logic)**:
    * *Input:* Loads the 140 metadata definitions (Yes/No, Dates, Entities, Clause Checks) from the Database.
    * *Process:* Utilizing the massive local H100 compute cluster, it brute-forces prompt batches across the chunks to pre-compute the answers for all 140 fields.
    * *Output:* Structured JSON stored in the `contract_metadata` PostgreSQL table.
4.  **`AdvancedEnrichmentStep`**:
    * **Multi-Phase Summarization:** Generates standard brief/detailed summaries for chunks. For documents exceeding 30k characters, it triggers a robust 4-phase Map-Reduce pipeline (Rank -> Extract -> Synthesize -> Critique).
    * **Intent-Based Summaries:** Pre-computes specialized page summaries based on user personas (e.g., Executive Summary, Technical Deep Dive, Troubleshooting Guide).
    * **Table Summarization:** Converts complex HTML pricing tables and SLAs into natural language paragraphs so the Vector DB can accurately retrieve structured data.
    * **HyDE (Hypothetical Document Embeddings):** Generates synthetic user questions for specific legal clauses to bridge the semantic gap during retrieval.
    * **Vision Analysis:** Uses vision models to extract keywords and descriptions from embedded document images.
    * **Knowledge Graph Extraction:** Maps entity relationships via triplets (Subject -> Predicate -> Object).
5.  **`EmbedAndStoreStep`**:
    * Stores vectors in the ChromaDB `legal_contracts` collection.

---

## 5. Application 3: Legal Document Retriever (LDR)

**Category:** (R) Retriever
**Role:** The "Hybrid Search Engine"
**Input:** Orchestrator Search Query (gRPC)
**Output:** Structured JSON containing SQL metadata and/or Vector citations.

The LDR is a dedicated gRPC service that acts as the bridge between the broader Orchestrator ecosystem and the Legal Knowledge Base. Because contract queries can be strictly factual ("What is the term?") or highly interpretive ("Summarize the liability limits"), the LDR uses a hybrid retrieval strategy.

### 5.1 Query Intent Routing
When passed a user query, a fast, small LLM (e.g., `gemma3:8b`) performs Intent Classification to route the search:

1.  **Metadata/SQL Intent:** If the query maps to one of the 140 pre-extracted fields (e.g., "Show me all contracts with Auto-Renew"), the LDR translates the query into a PostgreSQL `SELECT` statement against the `contract_metadata` table.
2.  **Semantic/Vector Intent:** If the query is open-ended or analytical (e.g., "How does the indemnification clause handle data breaches?"), the LDR converts the query to an embedding and searches ChromaDB.
3.  **Hybrid Intent:** If the query requires both (e.g., "Summarize the SLA penalties for contracts expiring in 2026"), the LDR filters by date in SQL, then performs a vector search only on the chunks belonging to that filtered subset of documents.

### 5.2 Context Formatting & Output
To prevent hallucinations in downstream synthesis, the LDR returns a strictly formatted payload:
* **SQL Hits:** Returned as clean key-value pairs representing the extracted fields.
* **Vector Hits:** Returned with the exact `clause_text`, the `document_name`, and a `citation_link` (e.g., `#page=4&chunk_id=102`) so the LDA UI can automatically scroll the user to the source material.
* **Reranking:** Vector results are passed through an NVIDIA NIM reranker to ensure only the highest-confidence clauses are returned.

---

## 6. Application 4: Legal Document Assistant (LDA)

**Category:** (A) Assistant
**Role:** The "Interface" (Ironclad Replacement)
**Input:** User Query / UI Interaction
**Output:** Split-Screen Metadata Dashboard + Cluster Remote Control

### 6.1 Architecture
Built as a modern React Single Page Application (SPA) using Feature-Sliced Design (FSD), served by a FastAPI backend. It serves as the primary interactive portal for Legal, Deal Desk, and Finance teams.

### 6.2 Core UI Features
* **Split View:** The left pane renders the original Document (PDF stream) or the AI-generated Markdown (with native HTML table rendering). The right pane displays the extracted metadata.
* **Validation Interface (HITL):** Users review the AI-extracted fields and logprob confidence scores. Editing a field automatically triggers a background save (`onBlur`) to PostgreSQL. 
* **CMD Resolution Autocomplete:** Unassigned documents can be quickly mapped to the correct corporate hierarchy using a hybrid caching/remote gRPC autocomplete component linked to the CMD Registry.
* **Active Cluster Remote Control:** The LDA is not just a passive reader. Using a dedicated gRPC connector (`ldp_connector`), users can click "Reprocess" or "Cancel" in the UI to instantly dispatch commands to the backend LDP worker nodes.
* **Smart Polling:** Utilizing React Query, the Document Explorer automatically polls the database every 2 seconds *only* when active processing jobs are detected, providing real-time pipeline status updates without refreshing.
---

## 7. Implementation Plan & Roadmap

Given that the core Parser (LDP) is largely complete and the Assistant (LDA) UI is functional, the remaining build is prioritized to deliver deep semantic search and self-service capabilities to the business.

### Phase 1: The "Read-Only" Dashboard (LDA) - [COMPLETE]
* **Objective:** Allow users to browse their contract repository and immediately see the AI's metadata extractions alongside the original document.
* **Deliverables:** * Built the core Split-Screen React UI and FastAPI backend.
  * Implemented the backend endpoints to fetch the parsed Markdown and the **foundational metadata** (Category, Customer, Date, Contacts) extracted by the LDP directly from PostgreSQL.

### Phase 2: Human-in-the-Loop (HITL) Integration - [COMPLETE]
* **Objective:** Establish trust by allowing legal professionals to verify, correct, and control the AI's work.
* **Deliverables:**
  * Added edit capabilities to the extracted fields in the UI with background auto-saving (`onBlur`).
  * Built the active gRPC remote control allowing users to trigger `Reprocess` or `Cancel` commands directly to the LDP cluster.
  * Implemented the hybrid CMD Autocomplete to seamlessly map unassigned documents to the official corporate hierarchy.

### Phase 3: Legal Document Ingestor (LDI) - [IN PROGRESS]
* **Objective:** Build the semantic engine that bridges the gap between raw text and searchable vector/SQL databases, fulfilling the requirements of Project CMDMeta.
* **Deliverables:**
  * **Semantic Chunking:** Develop a robust, fault-tolerant chunking strategy. Because relying on Markdown headers fails across decades of inconsistent legal formatting, the system uses an LLM with sliding windows to extract precise clause pointers, backed by Python logic to sanitize boundaries and stitch tables.
  * **The "140 Fields" Map-Reduce:** Utilize the local H100 compute cluster to execute prompt batches across the chunks, pre-computing and extracting the "Static Answers" (Metadata) for all 140 required fields into PostgreSQL.
  * **Advanced Semantic Enrichment:** Implement a massive enrichment pipeline including:
      * **Map-Reduce Summarization:** A 4-phase pipeline to accurately summarize massive 200-page MSAs.
      * **Intent Summaries:** Pre-computing Executive and Technical summaries to speed up real-time RAG.
      * **Table-to-Text:** Converting HTML pricing tables into natural language for vector search.
      * **HyDE:** Generating synthetic questions for complex clauses.
  * **Vector Generation:** Generate embeddings for the chunks, summaries, and questions, storing them in the ChromaDB `legal_contracts` collections.

### Phase 4: The Hybrid Retriever (LDR) - [PLANNED]
* **Objective:** Enable complex, conversational Q&A against the contract repository with precise, clickable citations.
* **Deliverables:**
  * Build the `gRPC` service that handles open-ended chat queries from the LDA UI.
  * Implement an **Intent Routing** layer (using a fast, small LLM) to dynamically route user queries to either PostgreSQL (for exact metadata hits across the 140 fields) or ChromaDB (for semantic clause matching).
  * Integrate NVIDIA NIM reranking to ensure only the highest-confidence clauses are returned to the user.
  * Connect the citations to the LDA UI to enable the "Auto-Scroll" feature, taking the user directly to the source text in the split-screen view.

### Phase 5: Dynamic Field Generation (Self-Service) - [PLANNED]
* **Objective:** Future-proof the platform by making the extraction engine entirely self-service for the business, allowing them to adapt to new legal requirements instantly.
* **Deliverables:**
  * Add a UI feature in the LDA allowing authorized Deal Desk/Legal users to define a "141st custom field" (e.g., *"Does this contract mention the new 2026 tax regulation?"*).
  * Build the backend logic to insert this new prompt into the database.
  * Develop an asynchronous Celery task that retroactively runs the new extraction prompt against the entire database of previously processed markdown chunks, automatically populating the new column across all historical contracts without requiring a full re-parse.

### Phase 6: The Collaborative Workflow Engine (CLM State Machine) - [PLANNED]
* **Objective:** Evolve the Legal Document Assistant (LDA) from a post-execution validator into a full-lifecycle Contract Lifecycle Management (CLM) system, replacing legacy platforms like Ironclad.
* **Deliverables:**
  * **Lifecycle State Machine:** Expand the PostgreSQL schema to track active negotiation states (e.g., `drafting`, `internal_legal_review`, `pending_counterparty`, `out_for_signature`, `executed`).
  * **Email-Driven Ingestion:** Adapt the Remittance Mail Ingestor (RMI) pattern to monitor a dedicated negotiations inbox (e.g., `negotiation@gtt.net`). The system will ingest incoming emails, associate them with active contract records, and seamlessly launch or update negotiations directly from Outlook.
  * **Approval Routing & Collaboration:** Build conditional workflow logic for internal approvals across departments, including UI capabilities for commenting, messaging, and assigning tasks.

### Phase 7: Automated Drafting & Redline Management - [PLANNED]
* **Objective:** Handle live, multi-party negotiations by safely processing and generating `.docx` files without compromising structural legal formatting.
* **Deliverables:**
  * **OpenXML Redline Processor:** Develop a specialized Python service (using `python-docx` or `lxml`) to programmatically diff OpenXML runs and paragraphs. This ensures the system can track changes and generate safe, cleanly formatted redline documents byte-for-byte.
  * **Semantic Version Diffing:** Integrate the reasoning LLM (e.g., `gemma3:27b`) to analyze the delta between V1 and V2 of a contract, automatically summarizing changes in risk profiles, liability caps, or SLA commitments for the Deal Desk.
  * **Context-Aware Email Generation:** Leverage the RAG capabilities to automatically draft intelligent, context-aware email replies to counterparties during the negotiation loop, replacing simple rule-based templates.

### Phase 8: E-Signature & Execution Boundary - [PLANNED]
* **Objective:** Cross the trust boundary by giving the SERA ecosystem the ability to legally bind the company and automatically execute finalized contracts.
* **Deliverables:**
  * **(C) DocuSign Connector:** Build a dedicated, reusable microservice within the SERA ecosystem to handle third-party e-signature APIs.
  * **Outbound Envelope Generation:** Enable the LDA to dispatch finalized PDFs, signer details, and tagging coordinates to the DocuSign API via a standard gRPC call.
  * **Inbound Webhook Listener (DocuSign Connect):** Expose a secure REST endpoint on the Sera AI Gateway to catch asynchronous signature events. Upon the `completed` event, the connector will automatically download the cryptographically sealed PDF to the `/assets/vault/` and trigger the state machine to advance to `executed`.

### Phase 9: Continuous Learning & Self-Healing RAG - [PLANNED]
* **Objective:** Create a zero-overhead Reinforcement Learning from Human Feedback (RLHF) loop. Instead of relying on expensive generalized fine-tuning, the system will crowdsource its accuracy directly from the Deal Desk and Legal teams using the platform.
* **Deliverables:**
  * **HITL Correction UI:** If the chatbot answers a question incorrectly (e.g., pulling the wrong termination clause), the user can browse the Split-Screen Document Explorer and flag the *correct* chunk that should have been retrieved.
  * **The Golden RAG Cache:** The system saves this interaction (the user's original query vector + the correct `chunk_id`) in a dedicated PostgreSQL table, strictly scoped to that specific CMD Client Group.
  * **Self-Healing Retrieval:** Modify the Legal Document Retriever (LDR) to check this cache first. If a future user asks a semantically similar question (passing a high fuzzy/vector similarity threshold against the cached query), the LDR will automatically force-inject the human-validated chunk into the LLM's context window, bypassing standard vector similarity failures.
  * **Result:** The RAG system becomes hyper-specialized and perfectly tuned to each specific customer's document quirks automatically over time.