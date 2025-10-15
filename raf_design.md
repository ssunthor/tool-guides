# /ai/assets/docs/tool_guides/raf_design.md
## Remote Automation Framerwork (RAF) - Design Document

**Version:** 0.5
**Date:** 2025-08-12
**Author(s):** Martin Kecka, Gemini AI
**Category:** Framework (F)
**Acronym:** RAF (Remote Automation Framework)

---

### Changelog (v0.4 to v0.5):

* **Architectural Shift:** RAF is refactored from a central automation "hub" with internal clients to a specialized "secure execution engine."
* **Scope:** RAF's primary responsibility is now to run pre-defined, version-controlled, multi-system workflows (e.g., "provision a new server"). Simple, single-system operations are now handled by external "Connector" modules.
* **Component Removal:** The `clients/` directory and the concept of internal, generic client libraries have been removed from RAF. Workflows are responsible for their own client instantiation.
* **Component Refactoring:** The "Helper Function Registry" is now the "Workflow Registry." The "Reporting Engine" is now considered a type of workflow and has been merged into the `automation/` module.
* **AI Interaction:** The AI (RAA) now discovers and triggers high-level workflows in RAF, rather than low-level helper functions.

---

## 1. Introduction & Goals

### 1.1. Purpose

The **Remote Automation Framework (RAF)** is a centralized and secure execution engine designed to run pre-defined, multi-system automation workflows and reports. It serves as the trusted backend for executing complex or sensitive operations that are too high-stakes for an AI to compose dynamically on the fly.

It is not a general-purpose automation platform. Instead, it is a specialized framework that runs audited, version-controlled scripts when requested by an authorized orchestrator, such as the Remote Automation Assistant (RAA).

### 1.2. Key Objectives

* **Secure Workflow Execution Environment:** Provide a sandboxed environment to run audited automation scripts, preventing direct, arbitrary command execution by AI agents.
* **High-Level Abstraction:** Expose complex business processes (e.g., "provision a new VM," "generate a quarterly security report") as single, callable API endpoints.
* **Modularity & Extensibility:** Design with a modular architecture that allows for the easy addition of new, self-contained automation workflows.
* **AI & User Operability:** Offer a robust gRPC API for AI systems (like RAA) and a CLI for human operators to discover and trigger these high-level workflows.
* **Reliability & Auditing:** Ensure that complex operations are performed consistently and that all actions are logged for security and compliance purposes.
* **Discoverability:** Enable AI systems to easily discover RAF's capabilities, including available workflows, their parameters, expected outputs, and procedural guides.

### 1.3. Relationship with RAA and Connectors

RAF is a critical component within a larger automation ecosystem, functioning as a specialized endpoint.

* **Remote Automation Assistant (RAA):** The "thinker" and primary orchestrator. It parses user requests, forms plans, and decides which tool to use.
* **Connectors (External):** Simple, single-system modules (e.g., a NetBox Connector, a CloudStack Connector). For straightforward tasks like "get a VM's IP from NetBox," the RAA calls the relevant Connector directly.
* **RAF:** The "doer" for complex tasks. When the RAA identifies a task that requires multiple systems, audited steps, or sensitive operations (e.g., "provision a full VM stack"), it delegates the entire task by calling a single, pre-defined workflow on RAF. The workflow script within RAF then handles all the necessary interactions with the end systems.

---

## 2. Core Architectural Principles

* **API-First:** All functionalities are exposed through a well-defined gRPC API centered on executing and managing workflows.
* **Workflow-Centric:** The core unit of work in RAF is the workflow. These are immutable, version-controlled scripts that cannot be altered at runtime by an external caller.
* **Principle of Least Privilege:** Callers (like the RAA) can only trigger pre-approved workflows. They cannot execute arbitrary code or access underlying systems directly through RAF.
* **Idempotency:** Workflows should be designed to be idempotent where applicable to ensure predictable outcomes even if re-run.
* **Statelessness:** RAF's API endpoints are stateless. The state of any long-running workflow is managed internally by a dedicated task management service.

---

## 3. Proposed RAF Architecture

### 3.1. Overall Structure (Conceptual Diagram)

+--------------------------------------------------------------------------------------------------+
|                                           Consumers                                              |
|            +-----------------------+  +-------------------+  +-----------------------+           |
|            |    SERA's RAA / LLM   |  |  Human Operator   |  | Cron Job / Scheduler  |           |
|            +-----------------------+  +-------------------+  +-----------------------+           |
+--------------------------------------------------------------------------------------------------+
|                   |                      |                               |                       |
|                   | "Asks 'What can you do?'" / "Triggers 'provision_vm'"| "Runs 'raf report generate...'" / "Triggers scheduled report"
|                   V                      V                               V                       |
+--------------------------------------------------------------------------------------------------+
|                                    Remote Automation API Layer                                   |
|            +------------------------------------+  +--------------------------------+            |
|            |        AutomationService API       |  |       ReportingService API     |            |
|            +------------------------------------+  +--------------------------------+            |
+--------------------------------------------------------------------------------------------------+
|                   |                      |        |                           |                 |
|                   | "Invokes script from"|        | "Reads from"              | "Reads from"    |
|                   V                      |        |                           V                 V
+------------------------------------+      |        +---------------------------+  +-------------------------+
|                                    |      |        |                           |  |                         |
|      automation/workflows/         |      |        |    Automation Registry    |  |     Report Registry     |
|                                    |      |        |         (DB)              |  |         (DB)            |
+------------------------------------+      |        +---------------------------+  +-------------------------+
|                   |                      |        | Invokes script from       |
|                   |                      |        V                           |
|                   |                      +------------------------------------+
|                   | "Executed by"               |    reporting/reports/      |
|                   +---------------------------->|                            |
|                   |                           | +---------------------------+
|                   V                           |
+-------------------------------------------------------------------+
|                                 Core Engine                       |
+-------------------------------------------------------------------+

**Internal Execution Flow:**

* Scripts from the **Automation Module** and **Reporting Module** are executed by the **Core Engine**.

### 3.2. Core Components

**RAF API Layer (`api` directory):**
* **Technology:** gRPC is the primary interface. A secondary FastAPI interface can be exposed for simple webhooks or browser-based API docs (OpenAPI).
* **Responsibilities:** Exposes two distinct, high-level gRPC services:
    * `AutomationService`: The exclusive interface for the RAA (AI) to discover and execute action-oriented workflows (`ExecuteAutomation`, `ListAvailableAutomations`).
    * `ReportingService`: The interface for human operators (CLI) and schedulers (cron) to manage and generate reports (`GenerateReport`, `ListAvailableReports`).

**RAF Core Engine (`core` directory):**
* **Responsibilities:** Receives requests from the API layer and invokes the correct internal workflow script from the `automation/` directory. Manages the execution context, injects necessary services (like logging and configuration), and oversees the task's lifecycle.

**Workflow Registry:**
* **Implementation:** A dedicated SQL database table managed by the `WorkflowRegistryService`.
* **Function:** Stores metadata for each available workflow (name, description, version, required permissions, and a parameter schema like JSON Schema). This registry is the source of truth for the `ListAvailableWorkflows` API call, enabling dynamic discovery by the RAA.

**Automation Engine (`automation` directory):**
* Contains logic for executing complex, multi-step automation workflows.
* **Workflow Definitions:**
    * **YAML:** For simpler, Ansible-like playbooks. Stored in `automation/playbooks/`.
    * **Python Scripts:** For more complex logic. Stored in `automation/workflows/` (e.g., `vm_provisioning.py`).
    * Reusable automation functions can also be placed in `automation/functions/`.
* Leverages client libraries and services. Exposed via the RAF API.

**Reporting Engine (`reporting` directory):**
* Generates reports by fetching data from various client libraries and processing it.
* `BaseReport` Class: An abstract base class (`reporting/base_report.py`).
* Individual reports will be modules in `reporting/reports/`.
* **Data Caching:**
    * Leverages `SQLCacheService` for input data.
    * Reports can specify cache validity or trigger cache refresh.
    * Allows multiple reports to use the same pre-cached data, beneficial for reports scheduled concurrently or for rapid testing/iteration by SERA.

**Service Layer (`services` directory):**
* `ConfigurationService`: Loads and provides secure access to RAF configurations (database connections, credentials for workflows to use, etc.) from `config.yaml`.
* `LoggingService`: Centralized, structured logging for all workflow executions.
* `SQLCacheService`: Provides persistent, queryable caching for workflows that need to store and reuse data.
* `WorkflowRegistryService`: Manages the CRUD operations and querying for the Workflow Registry.
* `TaskManagementService`: (Future) Manages asynchronous, long-running workflows.

**CLI Interface (`raf.py`):**
* The main entry point, using a library like Typer. It acts as a gRPC client to the RAF API, with distinct command groups for interacting with the AutomationService and ReportingService (e.g., `raf automation execute ...` and `raf report generate ...`).

---

## 4. Proposed Directory & File Structure

/ai/raf/
├── raf.py                      # Main application entry point (includes CLI)
├── bootstrap_helper.py         # For venv setup
├── requirements.txt
├── alembic.ini                 # Alembic config for DB migrations
│
├── api/                        # API layer (gRPC services, optional FastAPI)
│   ├── grpc_server.py
│   ├── rest_main.py            # (Optional) FastAPI app
│   ├── dependencies.py         # Common dependencies
│   └── services/               # gRPC service implementations
│       ├── init.py
│       ├── clients_service.py
│       ├── automation_service.py
│       ├── reporting_service.py
│       └── admin_service.py
│   └── protos/                 # Protobuf definitions
│       └── raf.proto
│
├── core/                       # Core engine, workflow logic
│   └── engine.py
│
├── services/                   # Internal RAF services (business logic)
│   ├── config_service.py
│   ├── logging_service.py
│   ├── cache_service.py
│   └── registry_service.py     # Manages the Workflow Registry
│
├── automation/                 # Automation engine
│   ├── init.py
│   ├── llm-automation-guide.md # LLM guide for using the automation engine
│   ├── playbooks/              # YAML playbook definitions
│   │   └── example_playbook.yaml
│   ├── workflows/              # Python workflow scripts
│   │   └── vm_provisioning.py
│   ├── functions/              # Reusable automation Python functions
│   └── scripts/                # Pre-defined, audited scripts for SSH execution
│       ├── devices/
│       │   └── get_interfaces.py
│       └── ai_scripts/           # For AI-generated scripts (sandboxed/reviewed)
│
├── reporting/                  # Reporting engine and report modules
│   ├── init.py
│   ├── llm-reporting-guide.md  # LLM guide for using the reporting engine
│   ├── base_report.py
│   ├── report_orchestrator.py
│   └── reports/
│       └── nb_cs_vm_discrepancy.py
│
├── models/                     # SQLAlchemy models for internal DB
│   ├── cache_models.py
│   └── registry_models.py      # Defines the 'workflows' table schema
│
├── config/
│   ├── config.default.yaml
│   └── config.yaml             # User's actual configuration
│
├── utils/                      # Common utility functions
├── alembic/                    # Alembic migration scripts
├── tests/                      # Unit and integration tests
└── docs/
    └── raf_design.md           # This file


---

## 5. Data Management & Configuration

* **Configuration (`/ai/raf/config/config.yaml`):** Stores RAF's internal settings and provides a secure vault for workflows to retrieve credentials (e.g., API keys, SSH keys) needed to access external systems. Managed by `services/config_service.py`.
* **Internal SQL Database (PostgreSQL):** Will use the existing `sera_ai_dev_postgres` instance with a new `raf_db` database. Managed by SQLAlchemy and Alembic. Stores the Workflow Registry, SQL Cache metadata, and audit logs.
* **Asset Storage (`/ai/assets/`):**
    * `/ai/assets/logs/raf/`: For structured application and workflow logs.
    * `/ai/assets/logs/raf/audit/`: For a dedicated, immutable audit trail.
    * `/ai/assets/cache/raf/`: For file-based caching by workflows.
    * `/ai/assets/tmp/raf/`: For temporary files generated by workflows.

---

## 6. AI Interaction & Discoverability

### 6.1. Formal Definitions

* **gRPC Service Definitions (Protobuf):** The RAA will use the `.proto` file to understand the `WorkflowService`, its methods (`ExecuteWorkflow`, etc.), and message structures. This is the primary contract.
* **Workflow Registry API:** The `ListAvailableWorkflows` gRPC method is the primary discovery mechanism. It returns a list of all registered workflows, their descriptions, and parameter schemas. The RAA uses this information to understand what high-level tasks it can delegate to RAF.

### 6.2. LLM Instruction Guides

* **Purpose:** A markdown file (`llm-workflow-guide.md`) provides narrative context to the RAA's LLM on how to choose between using a simple Connector and delegating to a complex RAF workflow.
* **Content Example:** "If the user asks to simply 'check the status of a VM', use the NetBox Connector. If the user asks to 'build a new VM for the web team', this requires multiple steps across CloudStack, NetBox, and DNS. Therefore, you should invoke the `provision_web_vm` workflow in RAF."
* **Access:** The RAA can fetch this guide's content via a simple RAF gRPC endpoint.

### 6.3. AI Interaction Pattern

The RAA does not generate code for RAF. The security model depends on RAF's workflows being human-written and audited. The interaction is as follows:

1.  RAA receives a complex user request.
2.  RAA consults its knowledge base, including the list of available RAF workflows and the LLM guide.
3.  RAA determines that the request matches the `provision_web_vm` workflow.
4.  RAA formulates a valid gRPC call to RAF's `ExecuteWorkflow` method with the correct workflow name and parameters.
5.  RAF executes the pre-defined, trusted workflow and returns the result to the RAA.

---

## 7. Security & Permissions Model

* **Authentication:** The gRPC API will use mTLS and token-based authentication to ensure requests only come from authorized services like the RAA.
* **Authorization (RBAC):** Permissions are applied at the workflow level. A user/service account (passed from RAA) is granted permission to execute specific workflows (e.g., `finance_team` role can run `generate_billing_report` but not `provision_vm`). This is checked by the Core Engine before invocation.
* **Credential Management:** Workflows do not contain hardcoded secrets. They request necessary credentials (e.g., a CloudStack API key) from the `ConfigurationService` at runtime, which stores them securely.
* **Execution Sandboxing:** Workflows that need to run shell commands (via SSH or Salt) will do so under strict controls:
    * They will primarily trigger pre-defined, audited scripts stored in `/ai/raf/automation/scripts/`.
    * Any direct command execution will be limited to a small, hardcoded whitelist within the workflow's code.
    * There is no mechanism for an external caller to pass arbitrary shell commands to a workflow.
* **Audit Logging:** All workflow executions are logged to `/ai/assets/logs/raf/audit/` with the initiating user context, timestamp, parameters, and outcome.

---

## 8. Phase 1 Development Plan & Example Use Case

### 8.1. Focus for Phase 1

* **RAF Core & API:** Setup the gRPC server with two distinct services: the `AutomationService` (for the RAA) and the `ReportingService` (for humans/cron). Implement the Core Engine, `ConfigurationService`, and `LoggingService`.
* **Registries:** Implement the SQL schemas and services for both the `AutomationRegistry` and the `ReportRegistry`.
* **Initial Report:** Implement the "VM Discrepancy Report" as the first task for the reporting module to prove the end-to-end architecture for human-triggered tasks.
* **CLI:** Implement basic CLI command groups for both modules (`raf automation ...` and `raf report ...`).
* **Internal DB:** Use the existing PostgreSQL instance on the development server.

### 8.2. Example Use Case: "VM Discrepancy Report"

This use case demonstrates how a human operator runs a report. It purposefully does not involve the RAA, which is unaware of the Reporting module.

1.  **Create Report Script** (`reporting/reports/nb_cs_vm_discrepancy.py`)
    * A developer writes the report script. Crucially, this script does not contain any low-level client libraries. It orchestrates calls to the external Connectors.
    * The script inherits from the `reporting.base_report.BaseReport` class.
    * It uses RAF's internal `ConfigurationService` to get the gRPC network addresses for the NetBox-Connector and CloudStack-Connector.
    * Using gRPC client stubs, it calls the high-level functions exposed by the Connectors:
        * `netbox_connector_stub.list_vms(fields=['name', 'primary_ip'])`
        * `cloudstack_connector_stub.list_virtual_machines(details=['name', 'ipaddress'])`
    * It receives the structured data back from both Connectors.
    * It compares the two lists of VMs to find discrepancies.
    * It formats the final output as a JSON object or a formatted table and returns it.

2.  **Register the Report**
    * The developer adds an entry to the `reports` table in RAF's database (likely via an Alembic migration script).
    * This entry includes `name: 'nb_cs_vm_discrepancy'`, a description, and defines any available parameters like `--output-format`. This makes the report discoverable by the `ReportingService`.

3.  **Expose via API & CLI**
    * The report is now automatically available through the `ReportingService`'s `GenerateReport` gRPC method.
    * The CLI provides a user-friendly way to invoke it: `raf report generate nb_cs_vm_discrepancy --output-format table`.

4.  **Execution by a Human Operator**
    * An infrastructure operator needs to check for inconsistencies.
    * They run `raf report list` to see available reports.
    * They then execute the command `raf report generate nb_cs_vm_discrepancy`.
    * RAF's `ReportingService` receives the request, invokes the script, which in turn calls the external Connectors, gets the data, and returns the final, processed report to the operator's console.

---

## 9. Open Questions & Discussion Points

* **Workflow Parameter Schema:** A formal schema (e.g., JSON Schema stored as `TEXT` in the DB) is needed for defining workflow parameters to enable robust validation and UI generation.
* **Credential Injection:** What is the most secure and flexible pattern for the `ConfigurationService` to provide credentials to a running workflow instance?
* **LLM Instruction Guide Access:** A simple gRPC endpoint in RAF to serve the markdown guide's content is the cleanest approach for future decoupling.
* **Task Management for Long-Running Operations:** Detailed design for asynchronous task management (e.g., using Celery or an internal queue) is deferred past Phase 1. For now, all workflows will be synchronous.
* **Dependency Management for Workflows:** How should Python dependencies unique to a single workflow be managed to avoid bloating the main `requirements.txt`? (Potential for isolated venvs per workflow, though this adds complexity).