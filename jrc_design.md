# Jira Connector (JRC) - Design Document

**Version:** 0.1.0
**Date:** 2026-05-04
**Author:** Martin Kecka & Gemini AI
**Category:** Connector (C)
**Acronym:** JRC

---

## 1. Introduction & Purpose

The **Jira Connector (JRC)** is a specialized gateway module within the SERA AI ecosystem. It serves as the deterministic, authoritative interface for all interactions with Atlassian Jira Cloud. 

Its primary purpose is to abstract away the immense complexity, bloat, and idiosyncratic rules of the Jira REST API (such as workflow transition limitations, GDPR-compliant accountId mappings, and ADF formatting) from the upstream AI Assistants. By providing a clean, LLM-optimized read/write interface, the JRC allows the Orchestrator to focus purely on semantic intent rather than syntactic execution.

**Key Design Decision (v2 vs. v3 API):** The JRC explicitly targets the Jira REST API **v2** instead of the newer v3. This is a deliberate architectural choice to avoid Jira's v3 "Atlassian Document Format" (ADF)—a deeply nested JSON AST that is highly prone to LLM hallucination. API v2 allows the platform to send and receive standard strings and Wiki Markup, which is significantly more reliable for AI generation.

---

## 2. Architecture & Integration

**Service Type:** External gRPC Service (Planned) / CLI Tool (Current MVP).

**Called by:** 
- Primarily the **Sera Orchestrator** or a specialized **Jira Assistant (JRA)** (Planned).
- Human users via the `jrc_cli.py` interface for testing, local data extraction, and offline schema discovery.

**Interacts with:**
- **Jira Cloud REST API v2:** The target system of truth. 
- **Internal Schema Files:** Utilizes a locally cached version of the Atlassian OpenAPI v2 schema for offline path discovery and dynamic routing.

---

## 3. Core Functionality & Process Flow

The JRC operates as a highly optimized, "Smart" Connector. It executes operations statelessly but contains hardcoded domain logic to handle Jira's specific API quirks.

### 3.1. LLM Context Optimization (The Cleaner)
Jira API responses are notoriously bloated (often returning 100+ `null` custom fields and deeply nested UI metadata per issue). To prevent context-window exhaustion in the LLMs, the JRC implements a strict `clean_issue` parser. 
- It strips out UI metadata (avatars, self-links, category IDs).
- It extracts and flattens highly valuable AI context: `subtasks`, `issuelinks` (crucial for dependency mapping), `comments`, `labels`, and locked date fields (`customfield_10015` -> `start_date`).

### 3.2. Smart Name-to-ID Resolution
Due to Atlassian's privacy changes, users cannot be assigned via name or email; an exact Atlassian `accountId` is required. The JRC encapsulates a `resolve_user_to_account_id` method that silently probes the `/user` and `/user/search` endpoints to translate natural language inputs ("Martin Kecka" or "martin.kecka@gtt.net") into the strict machine IDs required for execution.

### 3.3. Workflow State Transitions
Jira forbids updating an issue's `status` via standard data `PUT` requests. The JRC abstracts this complexity via the `transition_issue` method. When asked to change a status, the JRC dynamically fetches the currently available workflow transitions for that specific issue, maps the target status to the correct internal transition ID, and posts it to the `/transitions` endpoint.

### 3.4. Auto-Labeling (Auditability)
To ensure complete transparency across the enterprise, the JRC automatically injects the `sera-ai` label into the payload of every `Create` and `Update` operation. This guarantees that any action taken by the AI is permanently identifiable via simple JQL queries.

---

## 4. CLI Definition (MVP) 📜

The service currently exposes a robust Command-Line Interface (CLI) via `click`.

### Schema & Metadata Methods (`jrc schema`)
- `fetch`: Downloads the official Jira Cloud v2 OpenAPI schema to the local `/discovery` directory.
- `find <keyword>`: Searches the locally cached schema to help developers/AI discover exact endpoint paths and parameters.

### Data Management Methods (`jrc issue`)
- `get <key> [--clean] [--with-children]`: Fetches a single issue. The `--clean` flag optimizes the output for LLM consumption.
- `search <jql> [--clean]`: Executes a JQL search and returns matching issues.
- `create-subtask <project> <parent_key> <summary>`: Generates a new subtask attached to a parent.
- `assign <key> <user_identifier>`: Assigns an issue utilizing the Smart Name-to-ID resolver.
- `transition <key> <status>`: Moves an issue through the Jira workflow.
- `comment-add <key> <body>`: Appends a new comment to an issue.
- `comment-delete <key> <comment_id>`: Removes a specific comment.

---

## 5. Configuration (`config/settings.yaml`)

The JRC is configured via its standard ecosystem YAML file:
- `grpc_server`: Reserved host and port (e.g., `50082`) for future gRPC binding.
- `target_system`: The target Jira Cloud URL (`/rest/api/2`) and Atlassian Basic Auth credentials (Username + API Token).
- `logging`: Multi-file logging configuration (`jrc.log`, `jrc_err.log`).

---

## 6. Dependencies

- **Python Libraries:** `requests`, `click`, `pyyaml`. The service relies strictly on standard HTTP REST requests rather than heavy third-party Jira wrappers to maintain absolute control over payloads and error handling.

---

## 7. Logging & Auditing

The service implements the SERA standard:
- **Application Logging:** Standard operational events are written to the `/ai/assets/logs/jira_connector/` directory.
- **Output Dumping:** For the CLI MVP, all `get` and `search` results are dumped cleanly into the local `/output` directory as formatted JSON files for easy inspection.
- **Transaction Logging:** The skeleton is in place to support granular JSON transaction logs for every gRPC request/response once the service is bound to the Orchestrator.

---

## 8. Future Enhancements & Roadmap

- **gRPC Service Implementation:** Upgrade the current CLI wrapper into a fully compliant SERA gRPC service (`jc_service.proto`). This will allow the Orchestrator's `ToolExecutor` to invoke these methods dynamically.
- **Architectural Safety Boundary Definition:** Establish where specific idempotency and safety logic resides. 
  - *Debate:* Should the "Read-Before-Write" safety check (e.g., verifying an assignee isn't already assigned before firing a PUT, or requiring a GET before a DELETE) be hardcoded into the JRC Python logic, or should the JRC remain a "dumb" execution engine while the upstream **Jira Assistant (JRA)** handles statefulness and Human-In-The-Loop (HITL) confirmation prompts?
- **Wiki Markup Preprocessor Integration:** Fully integrate the Python `JiraPreprocessor` to seamlessly translate standard Markdown generated by the LLM into Atlassian's proprietary Wiki Markup (handling `{code}`, `{panel}`, and text styling automatically).
- **Dynamic Custom Field Resolution:** Build a discovery cache similar to the NetBox Connector that automatically maps human-readable custom field names (e.g., "Start Date") to their Jira-specific internal IDs (e.g., `customfield_10015`), shielding the LLM from arbitrary numeric keys.