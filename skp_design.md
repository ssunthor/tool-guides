SERA Knowledge Platform (SKP) - Design Document

**Version:** 0.0.1 (Draft)
**Date:** 2026-01-21
**Author(s):** Martin Kecka, Gemini AI
**Category:** Application (A) / Platform
**Acronym:** SKP

---

## 1. Introduction & Vision

The **SERA Knowledge Platform (SKP)** is a purpose-built, AI-native documentation system designed to replace Confluence as the central repository for enterprise knowledge.

Unlike traditional wikis which store data as unstructured HTML blobs ("Pages"), the SKP is built on a **Block-Based Architecture**. Every paragraph, code snippet, image, and table is an atomic, addressable unit of data. This structure allows the system to shift from a **Passive Ingestion Model** (periodically scraping external data) to an **Active Knowledge Model**:

1.  **Born-Structured Data:** Content is created as structured JSON blocks, eliminating the need for complex HTML parsing.
2.  **Real-Time Enrichment:** As users type, background AI processes immediately generate embeddings, extract knowledge triplets, and flag ambiguity.
3.  **The Orchestrator as a First-Class Author:** The SKP exposes a full API, allowing the SERA Orchestrator to autonomously create runbooks, log automation results, and draft articles based on user conversations.

---

## 2. User Experience (UX)

The SKP is a modern Single Page Application (SPA) built with **React**, mirroring the design language of the CAP and CAA tools.

### 2.1 The Editor Experience
The core of the SKP is a "Notion-style" block editor powered by **Tiptap**.
* **Slash Commands:** Users type `/` to insert headers, code blocks, or AI components.
* **Drag-and-Drop:** Blocks can be rearranged easily.
* **AI Co-Pilot:** A context-aware sidebar allows users to highlight a block and ask the AI to "Rewrite for clarity," "Expand this bullet point," or "Generate a table from this text."

### 2.2 The "Living" Knowledge Graph
Instead of static links, the SKP encourages **Entity Linking**. When a user types "@", they link not just to another user, but to defined concepts (e.g., `@Service:NetBox`, `@Server:uk-lon-01`). This explicitly builds the Knowledge Graph that the Orchestrator uses for reasoning.

### 2.3 Consumption & Search
* **Semantic Search:** Because every block is embedded, search results highlight the specific *paragraph* that answers a query, not just the whole page.
* **Dynamic Views:** Users can view documentation as standard text or switch to a **Graph View** to see how systems and concepts depend on each other.

---

## 3. Architecture & Stack

* **Frontend:** React + Vite + Material UI (Matches CAP/CAA).
* **Editor Engine:** **Tiptap** (Headless, creates JSON output).
* **Backend:** Python **FastAPI**.
* **Async Processing:** **Redis** + **Celery** (Crucial for non-blocking AI operations).
* **Data Stores:**
    * **PostgreSQL:** Stores the hierarchical page structure and block content (`JSONB`).
    * **ChromaDB:** Stores vector embeddings for every block.
* **Auth:** **Sera AI Gateway** (Header-based authentication).

---

## 4. Data Model: The Block Paradigm 🧱

The shift from "Pages" to "Blocks" is the defining architectural feature.

### 4.1 Database Schema (Simplified)

**`wiki_pages`**
* `id`: Integer (PK)
* `title`: String
* `slug`: String (URL friendly)
* `parent_id`: Integer (Adjacency List for hierarchy)
* `space_key`: String (e.g., 'SYSENG', 'DEV')
* `created_at`: Timestamp
* `updated_at`: Timestamp

**`wiki_blocks`**
* `id`: UUID (PK)
* `page_id`: FK to `wiki_pages`
* `block_type`: String (`paragraph`, `heading`, `code`, `image`, `table`)
* `content_json`: JSONB (The Tiptap structured node)
* `plain_text`: Text (Extracted for fast lexical search)
* `order_index`: Integer (Position on page)
* `embedding_id`: String (Link to ChromaDB)
* `last_processed_at`: Timestamp

**`knowledge_triplets`**
* `source_block_id`: FK to `wiki_blocks`
* `subject`: String
* `predicate`: String
* `object`: String

---

## 5. The "Active" Processing Pipeline ⚙️

When a user saves a page (or auto-save triggers), the backend does **not** simply dump text into a database. It triggers an intelligent, asynchronous pipeline via Celery.

### Step 1: Diffing & Change Detection
The backend compares the incoming JSON tree with the existing DB state. It identifies:
* **New Blocks:** Need full processing.
* **Updated Blocks:** Need re-processing (embedding/triplets).
* **Unchanged Blocks:** Ignored (saving massive compute resources).

### Step 2: Parallel Enrichment (Celery Tasks)
For every modified block, the following tasks are dispatched:

* **`tasks.embed_block`:**
    * Calls the AI Factory (NIM) to generate a vector embedding for the block's text.
    * Upserts the vector to ChromaDB with metadata (`page_id`, `block_type`).
* **`tasks.extract_triplets`:**
    * If the block contains factual assertions, calls a small LLM (`gemma3:4b`) to extract Subject-Predicate-Object triplets.
    * Updates the `knowledge_triplets` table.
* **`tasks.generate_hyde`:**
    * (Page Level) If substantial changes occurred, regenerates Hypothetical Questions (HyDE) for the page to improve retrieval.

### Step 3: Image Analysis
If an `image` block is added:
* **`tasks.analyze_image`:**
    * The backend passes the image to the Vision Model (`gemma3:27b` via vLLM/NIM).
    * The model returns a textual description and keywords.
    * This description is stored in the block's metadata and is **embedded**, making the image searchable by its content.

---

## 6. The "Smart Migration" Strategy 🚚

We will not manually copy-paste from Confluence. We will leverage the existing **Confluence Ingestor (CFI)** logic to perform an **AI-Assisted Refactor**.

### The Migration Pipeline (`migrate_confluence.py`)

1.  **Extract:** Use `ConfluenceClient` to fetch the raw storage format of a legacy page.
2.  **Clean:** Use `HtmlProcessor` to strip boilerplate macros.
3.  **Refactor (The AI Agent):**
    * Instead of saving the text, the script sends the content to `gemma3:27b`.
    * **Prompt:** *"You are a Technical Writer. Convert this unstructured HTML into a list of structured JSON blocks for Tiptap. Break long paragraphs. Extract code into code blocks. Ensure active voice."*
4.  **Load:** The script inserts the resulting JSON blocks directly into the `wiki_blocks` table.

**Result:** The documentation is not just moved; it is modernized, structured, and tagged during the migration process.

---

## 7. Integration with SERA Orchestrator

The SKP exposes two interfaces for the Orchestrator.

### 7.1 The Retriever Tool (`wiki_retriever`)
* **Function:** Replaces the `rag_confluence_retriever`.
* **Mechanism:** Performs hybrid search (Vector + Keyword) against the SKP database.
* **Advantage:** Returns highly specific **Blocks** rather than large text chunks, reducing token usage and increasing precision.

### 7.2 The Writer Tool (`wiki_writer`)
* **Function:** Allows the Orchestrator to create or update content.
* **Capabilities:**
    * `create_page(title, space, content_blocks)`
    * `append_log(page_id, log_entry)`
* **Use Cases:**
    * **Runbooks:** When RAF executes a task, it logs the output directly to a "Daily Operations Log" page.
    * **Knowledge Capture:** When SERA solves a unique problem in chat, it can offer: *"Shall I save this solution to the 'Troubleshooting' wiki space?"*

---

## 8. Roadmap & Development Plan

### Phase 1: The Foundation (MVP)
* [ ] Set up FastAPI backend with PostgreSQL/ChromaDB connections.
* [ ] Build basic React UI with Tiptap editor.
* [ ] Implement "Save" endpoint that saves JSON blocks to `wiki_blocks`.
* [ ] Implement basic `embed_block` Celery task.

### Phase 2: The Migration Engine
* [ ] Adapt `Confluence Ingestor` logic to feed the `wiki_blocks` table.
* [ ] Create the "AI Refactor" prompt for converting HTML to Tiptap JSON.
* [ ] Run migration on a sub-space (e.g., `NETBOX_PROVISIONING`) to test.

### Phase 3: The Active Intelligence
* [ ] Implement `extract_triplets` and `analyze_image` background tasks.
* [ ] Build the "Graph View" in the UI to visualize links.
* [ ] Integrate the `wiki_retriever` tool into the Orchestrator.

### Phase 4: Full Ecosystem Integration
* [ ] Implement the `wiki_writer` tool.
* [ ] Create "Self-Documenting" workflows in RAF that write to SKP.
* [ ] Decommission Confluence.

## 9. Export & Publishing Engine 🖨️

A core feature of SKP is its ability to transform internal documentation into polished, stakeholder-ready formats (PDF/DOCX) on demand.

### 9.1 Intent-Driven Generation
Unlike a standard "Print to PDF" which dumps the raw page, SKP uses an **AI Transformation Step**.
* **Internal Source:** The raw wiki page (e.g., "VM Provisioning - SysEng Internal").
* **Target Audience:** The user selects an intent (e.g., "Customer Guide", "Executive Summary", "Audit Report").
* **AI Transformation:**
    * The backend sends the blocks to `gemma3:27b`.
    * **Prompt:** *"Rewrite this technical documentation for a non-technical customer. Remove internal IP addresses. Expand acronyms. Change tone to professional/service-oriented."*
    * **Output:** A temporary stream of transformed JSON blocks.

### 9.2 Rendering Pipeline
* **PDF Engine:** Uses `WeasyPrint`.
    * Converts the (AI-transformed) JSON blocks into a clean HTML/CSS intermediate format.
    * Renders pixel-perfect PDFs with branded headers, footers, and cover pages.
* **DOCX Engine:** Uses `python-docx`.
    * Maps JSON blocks to Word styles (Heading 1, Normal, Code).
    * Generates editable documents for stakeholders who require Word format.

### 9.3 API Endpoints
* `POST /api/pages/{id}/export`
    * **Payload:** `{ "format": "pdf", "style": "customer_guide" }`
    * **Response:** Binary file stream (PDF/DOCX).