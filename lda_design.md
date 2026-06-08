# Legal Document Assistant (LDA) - Design Document

**Version:** 0.2.3
**Date:** 2026-06-08
**Author(s):** Martin Kecka, Gemini AI
**Category:** Assistant (A)
**Acronym:** LDA

---

## 1. Introduction & Purpose

The **Legal Document Assistant (LDA)** is the user-facing web application and API gateway for the broader Legal Knowledge Base (LKB) suite. Positioned as the interactive portal for Legal, Deal Desk, and Finance teams, it provides a modern, fast, and intuitive interface to explore, verify, and manage the lifecycle of AI-parsed legal contracts.

Acting as the "Interface" of the suite, the LDA decouples the user experience from the heavy backend processing of the Legal Document Parser (LDP). It reads directly from the shared PostgreSQL database to display real-time extraction results and provides a "Human-in-the-Loop" (HITL) validation interface. Furthermore, it acts as a remote control for the processing cluster, utilizing a dedicated gRPC connector to dispatch reprocess or cancellation commands directly to the LDP worker nodes.

---

## 2. Architecture & Core Components

The LDA is built using a modern decoupled architecture: a robust **FastAPI backend**, a **gRPC Telemetry Server**, and a highly responsive **React Single Page Application (SPA)** built with Vite and Material UI (MUI).

### 2.1 Backend Services & CLI
* **`cli/lda_cli.py`**: The central execution orchestrator built with `click`. Manages the startup of the REST API, the gRPC server, and provides local health-check commands (`module check-health`).
* **`rest_api/main.py`**: The API entry point. Configures CORS, custom logging middleware, serves the static React build, and mounts the API routers.
* **`rest_api/routers/documents.py`**: The core API endpoints. Handles listing documents (featuring server-side grouping and pagination), fetching detailed metadata with linked extracted clauses, streaming source PDFs, and processing HITL `PUT` requests to update fields. It also exposes a highly optimized `/dashboard-stats` endpoint for UI aggregations.
* **`rest_api/routers/docs.py`**: The documentation aggregator. Dynamically parses `settings.yaml` to read and serve raw markdown guides and YAML changelogs from upstream modules (LDP, LDI, LDR).
* **`grpc_api/services.py`**: The SERA Standard telemetry server. Runs on port 50064 to expose `GetAppInfo` and `GetDeepHealth` RPCs to the broader enterprise ecosystem for centralized monitoring.
* **`core/connectors/ldp_connector.py` & `cmc_client.py`**: The gRPC bridges. Establish secure channels to the LDP (port 50061) for job control, and the CMC (port 50053) for authoritative client entity resolution.

### 2.2 Frontend (React / Vite)
Structured using **Feature-Sliced Design (FSD)** principles, ensuring high maintainability:
* **`ui/src/pages/`**: The top-level route containers (`DashboardPage`, `DocumentExplorerPage`, `DocumentationPage`, `GlobalSearchPage`).
* **`ui/src/features/`**: Domain-specific UI components grouped by feature.
    * **`dashboard/`**: Contains `StatCard` and the `ParsingPerformanceWidget` for visualizing cluster metrics.
    * **`explorer/`**: Contains the complex list views with server-side filtering, clause filter sidebars, dynamic alphabetical accordions (`DocumentGroupAccordion`), and confidence tooltips.
    * **`document/`**: Contains the core Split-Screen UI. `DocumentViewer` toggles between the original PDF stream and the rendered Markdown with fuzzy regex text highlighting. `MetadataPanel` provides the editable forms for HITL validation, featuring the `CmdGroupAutocomplete`.
* **State Management**: Utilizes `@tanstack/react-query` for smart-polling and caching, ensuring the UI remains perfectly synced with the database without manual refreshes.

---

## 3. User Workflows

### 3.1 Real-Time Dashboard & Monitoring
Upon login, users land on the Dashboard. It queries a highly optimized backend endpoint (`/api/documents/dashboard-stats`) to display live KPIs: Total Processed, Success/Failure rates, Active Queue size, and unique entity counts. The Parsing Performance Widget gives technical users insight into the health and speed of the backend LDP cluster.

### 3.2 Document Exploration & Smart Polling
The Document Explorer provides a densely packed, filterable view of all contracts.
* **Server-Side Grouping:** Documents are instantly aggregated and filtered in PostgreSQL, categorized by their resolved `cmd_client_group_name` into dynamic alphabetical buckets (e.g., 'A - C', 'D - E') for scalable performance.
* **Smart Polling:** If any document in the list is in a `pending` or `processing` state, React Query engages a 2-second polling interval. Users can watch the progress bar fill and read live status updates streamed directly from the LDP Celery workers.
* **Confidence Tooltips:** Granular AI confidence scores (derived from token logprobs) are hidden behind an intuitive tooltip on the global confidence badge.

### 3.3 Global Full-Text Search
A persistent global search bar allows users to deep-search the actual text of parsed legal clauses across the entire repository. Clicking a search result automatically routes to the specific page, scrolls to the exact paragraph, and highlights the search term in both the parsed Markdown and the original PDF.

### 3.4 The Split-Screen Validation (HITL)
Clicking a document opens the unified `DocumentDetails` Split-Screen Viewer.
* **Left Pane (Viewer):** Uses `react-pdf` to stream the original source document, or `react-markdown` to render the fully stitched, table-repaired Markdown output generated by the AI.
* **Right Pane (Metadata):** Displays the extracted fields. Users can review the AI's reasoning, verify the resolved CMD Client IDs, and ultimately change the `Review Status` from `unassigned` to `validated`. 
* **Remote Control:** If the OCR was poor or the LLM hallucinated, users can click "Reprocess" to instantly fire a gRPC command to the LDP cluster to wipe the stats and re-run the pipeline.

### 3.5 Ecosystem Documentation & Release Notes
The `DocumentationPage` acts as a centralized knowledge base for the entire LKB suite. It dynamically fetches the system architecture documents and parses the YAML changelogs for the active modules, rendering them into an interactive, filterable timeline so users can track what features the AI cluster is currently supporting.

---

## 4. Key Technical Features

* **Dual-Service Architecture:** Runs both a FastAPI REST server for the UI and a separate SERA gRPC server for enterprise telemetry without thread blocking.
* **gRPC Remote Control:** Actively commands the LDP cluster using Protobuf-defined RPCs to manage job states, bypassing the need for complex database state-machine hacks.
* **React Query Smart Polling:** Minimizes database load. The frontend only hammers the backend for updates when it inherently knows a background task is actively running.
* **Dynamic Markdown Rendering:** Utilizes `remark-gfm` and `rehype-raw` to safely render complex HTML tables natively inside the Markdown view, fully supporting the LDP's "Double-Rip" table repairs.
* **Unified Theme Context:** A custom MUI theme provider featuring a modern glassmorphic dark mode, switchable via a context toggle that syncs with upstream Gateway headers.

---

## 5. Configuration & Deployment

* **`config/settings.yaml`**: The single source of truth for the environment. Maps the API port (7892), gRPC port (50064), database DSNs, target downstream connectors, and documentation file paths.
* **`legal_document_assistant.py`**: The primary executable. Routes to `cli/lda_cli.py` to initiate services (`python legal_document_assistant.py start-api-server`).
* **`deploy.sh`**: A shell script automating the Vite production build (`npm run build`) and the systemd service restart for seamless deployments.
* **`utils/logging_setup.py`**: Configures dual-channel logging (Console + Rotating Files) and actively filters out noisy API Gateway health checks (`/health`) from the access logs.

---

## 6. Future Enhancements

### 6.1 Chat & Semantic Retrieval Integration
**Objective:** Evolve the LDA from a metadata validator into a conversational RAG interface.
* **Implementation:** The UI will introduce a Chat interface pane. The backend will integrate with the future **Legal Document Retriever (LDR)** gRPC service to route natural language queries into Hybrid SQL/Vector searches, returning precise citations that auto-scroll the PDF viewer.

### 6.2 Dynamic Field Generation (Self-Service)
**Objective:** Allow business users to define new extraction fields without developer intervention.
* **Implementation:** Add a configuration UI allowing users to define a new prompt and data type. The LDA backend will insert this into the database and fire a broadcast command to the LDP cluster to retroactively run the new extraction against the cached Markdown of all previously processed documents.