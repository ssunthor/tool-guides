# FortiBleed Analyzer (FBA) - Design Document

**Version:** 0.2.0
**Date:** 2026-06-25
**Author:** Martin Kecka & Gemini AI
**Category:** Analyzer (A)
**Acronym:** FBA

---

## 1. Introduction & Purpose

The **FortiBleed Analyzer (FBA)** is a specialized security auditing tool designed to proactively scan and analyze Fortinet firewall configuration files for critical remote management vulnerabilities. 

Given the recent surge in "Stolen Fortinet Credentials" exploits, FBA targets misconfigurations that expose administrative interfaces (SSH, HTTPS) to the public internet without adequate access controls. It automates the parsing of massive configurations, extracting relevant security policies, and leverages a massive 31-Billion parameter LLM (Gemma-4-31B-IT) to intelligently evaluate FortiOS-specific edge cases, such as implicit ACCEPT behaviors in Local-In Policies.

FBA provides a "Single Pane of Glass" to security engineers, offering a comprehensive inventory dashboard, split-screen raw configuration viewing, and real-time LLM reasoning to validate and remediate vulnerabilities.

---

## 2. Architecture & Core Components

FBA is built on a decoupled, asynchronous microservice architecture designed to handle thousands of heavy firewall configs without blocking the user interface.

### 2.1 Backend Services & CLI
* **`cli/fba_cli.py`**: The unified CLI entry point. Orchestrates the FastAPI server, the Celery workers, and provides direct task control commands (e.g., `analyze`, `analyze-file`, `queue-status`, `queue-purge`).
* **`rest_api/main.py`**: The FastAPI application. Exposes REST endpoints for inventory pagination (`/api/inventory`), aggregated customer statistics (`/api/customers`), raw file streaming (`/api/inventory/{id}/config`), metadata filtering (`/api/filters`), and documentation serving (`/api/docs`).
* **`core/database/`**: Dedicated directory containing SQLAlchemy models and database connection pooling, ensuring better modularity.
* **`core/analyzer.py`**: The core extraction logic. Isolates and condenses relevant blocks (`config system interface`, `config firewall local-in-policy`, `config system admin`) from multi-megabyte configs into lightweight snippets suitable for the LLM context window. It also natively parses the firmware version and model directly from the configuration text prior to dispatching tasks.
* **`core/worker_celery.py`**: The asynchronous execution engine. Registers the `analyze_file_task` to communicate with the vLLM client, handling rate limits, timeouts, and JSON parsing.
* **`prompts/analyzer_prompt.py`**: A versioned Python module containing the LLM prompt, ensuring strict `prompt_version` tracking in the PostgreSQL database.

### 2.2 Frontend (React / Vite)
* **`ui/src/pages/DashboardPage.jsx`**: Provides an aggregated view of total vulnerabilities, success rates, and queue health.
* **`ui/src/pages/InventoryPage.jsx`**: The core exploratory interface. Features a Dual-View system (Paginated Grid vs. Alphabetical Customer Accordion) and dynamic status, model, and version filtering via dynamically loaded database metadata. The grouped Customer Accordion UI prominently displays device model and firmware version chips on individual firewall rows.
* **`ui/src/pages/DocumentationPage.jsx`**: A centralized knowledge base rendering this design document and the interactive FBA Release Notes (Changelog) within a dedicated UI tab.

### 2.3 Storage & Infrastructure
* **PostgreSQL (`fortibleed_analyzer_db`)**: Acts as the persistent source of truth for the UI. Caches all LLM findings, reasons, and remediation scripts, completely decoupling the UI from the LLM inference latency. It dynamically tracks `prompt_version`, `device_model`, and `device_version` to support comprehensive querying.
* **Redis (Localhost DB 9 & 10)**: Serves as the high-speed task broker and result backend for Celery. FBA utilizes a local Redis instance to circumvent complex, restrictive ACL issues encountered in the enterprise Valkey Sentinel cluster.

---

## 3. User Workflows

### 3.1 Synchronous File Analysis (CLI)
Engineers can directly test the LLM logic on isolated configuration files using the CLI:
`python3 fortibleed_analyzer.py analyze-file IPCNER-CHIC-1916844.conf`
This bypasses Celery and the database, streaming the exact LLM prompt and response directly to the console for rapid prompt engineering and validation.

### 3.2 Asynchronous Batch Processing
Using the `analyze --override` CLI command, the system iterates over the entire `/input` directory, spawning a stub in the PostgreSQL database (Status: Pending) and dispatching a Celery task. The logic natively prioritizes already-processed database entries, automatically dispatching them to the front of the Celery queue for rapid reprocessing. The Celery workers independently pick up the tasks, query the vLLM cluster, and write the findings back to the database.

### 3.3 Findings & Remediation (UI)
Security operators utilize the Web UI to filter for `vulnerable` devices. Clicking on a record opens a dedicated Dialog overlay. 
* **Left Pane**: Displays the raw, original Fortinet `.conf` file.
* **Right Pane**: Displays the LLM's explicit reasoning (e.g., "Missing trusthost ACLs on WAN interface") and provides the precise FortiOS CLI commands required to remediate the vulnerability.

---

## 4. Key Technical Features

* **FortiOS Logic Emulation**: FBA is specifically prompted to understand complex FortiOS quirks, such as the fact that `config firewall local-in-policy` catch-all rules lacking an explicit `set action deny` actually default to an implicit **ACCEPT**, rendering the device vulnerable despite the presence of a policy.
* **Smart Config Condensation**: Rather than stuffing entire 300,000-byte configuration files into the LLM, FBA parses the FortiOS tree structure to isolate only the exact interfaces, ACLs, and global settings relevant to remote management, drastically reducing inference latency and hallucination rates.
* **Resilient Authentication**: The UI seamlessly integrates with the GIA Gateway, dynamically reading `X-Forwarded-User` headers to resolve identities and provide personalized Theme support natively.
