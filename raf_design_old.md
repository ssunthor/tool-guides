 Remote Automation Framework (RAF)

**Last updated:** May 25, 2025
**Version:** 0.4
**Date:** 2025-05-24
**Author(s):** Gemini AI & Martin Kecka

---

## 1. Introduction & Goals

### 1.1. Purpose

The Remote Automation Framework (RAF) is envisioned as a centralized Python-based platform designed to streamline, manage, and execute a wide range of infrastructure automation tasks. It will serve as the primary backend for interacting with various systems like NetBox, CloudStack, Zabbix, and others, providing a unified interface for both human operators and AI-driven orchestrators like SERA.

### 1.2. Key Objectives

* **Centralized Automation Hub:** Provide a single point of control and execution for diverse automation scripts and workflows.
* **Modularity & Extensibility:** Design with a modular architecture that allows for easy addition of new client integrations (for different systems) and new automation capabilities.
* **AI & User Operability:** Offer robust APIs for AI systems (like SERA) and a user-friendly CLI for human operators and scripting.
* **Standardization:** Promote standardized ways of interacting with infrastructure components, logging, configuration, and error handling.
* **Security & Control:** Implement a granular permissions model to control what actions can be performed by users and AI, especially for sensitive operations.
* **Discoverability:** Enable AI systems to easily discover RAF's capabilities, including available functions, their parameters, expected outputs, and procedural guides for common tasks.
* **Foundation for Complex Operations:** Serve as the execution engine for multi-step provisioning processes, reporting, and inventory management.

### 1.3. Relationship with SERA Orchestrator

RAF will be a distinct application, initially co-located with SERA on the same server (in `/ai/raf/`) for performance and simplified initial deployment.

SERA will primarily interact with RAF via its API (gRPC preferred). This ensures decoupling and allows RAF to potentially serve other clients or be scaled independently in the future.

RAF will handle the direct execution of tasks, while SERA will focus on higher-level reasoning, planning, and user interaction.

## 2. Core Architectural Principles

* **API-First:** RAF's functionalities will be exposed through a well-defined API (gRPC for SERA, potentially REST for other tools/CLI). The CLI will be a consumer of this API.
* **Service-Oriented:** Core functionalities like caching, logging, and configuration will be treated as internal services.
* **Client Abstraction:** Interactions with external systems (NetBox, CloudStack, etc.) will be through dedicated, standardized client libraries.
* **Declarative & Imperative Capabilities:** Support both declarative definitions of desired states and imperative execution of specific commands/tasks.
* **Idempotency:** Strive for idempotency in automation tasks where applicable to ensure predictable outcomes.
* **Statelessness (where possible):** RAF API endpoints should be stateless where possible, with necessary state managed by dedicated services (e.g., task management, caching).

## 3. Proposed RAF Architecture

### 3.1. Overall Structure (Conceptual Diagram)

```text
+-------------------------------------------------------------------+
| SERA (AI Orchestrator) / Human User (CLI) / Other API Consumers   |
+---------------------+-----------------------+---------------------+
                      | (gRPC API Calls / CLI Commands) |
                      v                       v
+---------------------+-----------------------+---------------------+
|                     RAF API Layer (gRPC Server / FastAPI)         |
|              (Authentication, Authorization, Routing)             |
+---------------------+-----------------------+---------------------+
                      |                       |
                      v                       v
+---------------------+-----------------------+---------------------+
|                      RAF Core Engine & Services                     |
| (Request Handling, Workflow Orchestration, Task Management,        |
|  Config Service, Logging Service, SQL Cache Service,               |
|  Helper Function Registry Service)                                 |
+---------------------+-----------------------+---------------------+
      |         |         |         |         |         |         |
      v         v         v         v         v         v         v
+-----------+ +---------+ +---------+ +-------+ +---------+ +-------+ +------+
| NetBox    | |CloudStack | | Zabbix  | | SSH   | | Redfish | | AD    | | Salt |
| Client    | | Client    | | Client  | | Client| | Client  | | Client| |Client| ...
+-----------+ +---------+ +---------+ +-------+ +---------+ +-------+ +------+
|                       RAF Client Libraries                        |
+-------------------------------------------------------------------+
      ^         ^         ^         ^         ^         ^         ^
      |         |         |         |         |         |         |
+-----------+ +---------+ +---------+ +-------+ +---------+ +-------+ +------+
| NetBox    | |CloudStack | | Zabbix  | | Target| | BMCs    | | DNS   | | Minions|
| API       | | mgmt API  | | API     | | Servers|           |       | |      |
+-----------+ +---------+ +---------+ +-------+ +---------+ +-------+ +------+
```

### 3.2. Core Components

* **RAF API Layer** (`api` directory):
    * **Technology:**
        * **gRPC:** Primary interface for SERA for performance and typed contracts. Protobuf definitions will be key.
        * **FastAPI (Optional):** Can be exposed alongside gRPC to provide a RESTful interface for broader compatibility (e.g., CLI, simple webhooks, other tools) and automatic OpenAPI documentation.
    * **Responsibilities:**
        * Expose all RAF functionalities.
        * Handle authentication (e.g., API keys for SERA, OAuth2 for REST if used) and authorization (RBAC).
        * Request validation and routing to the RAF Core Engine.
        * Provide service definitions (Protobuf) and potentially an OpenAPI specification if FastAPI is used.
* **RAF Core Engine** (`core` directory):
    * **Responsibilities:**
        * Orchestrate internal operations.
        * Manage task execution (potentially with a queue for long-running tasks).
        * Interact with various services (Config, Logging, Caching, Helper Registry).
        * Invoke appropriate client library methods or automation/reporting engine functions.
* **Client Libraries** (`clients` directory):
    * **Design Philosophy:** Each client will follow a two-tier approach:
        * **Universal Base Client:** Provides low-level, generic methods to interact with any valid API endpoint of the target system (e.g., `request(method, endpoint, params, data)`). Handles authentication, base URL, pagination, and basic error handling. Includes configuration for static path components or default filters (e.g., `NETBOX_VM_API_PATH`, `NETBOX_VM_FILTERS`).
        * **Higher-Level Helper Functions:** Built on top of the Base Client. Provide semantic, user-friendly, and AI-friendly methods for common operations (e.g., `netbox_client.get_vm_by_name(name)`, `cloudstack_client.deploy_vm(params)`).
    * **Helper Function Registry:**
        * Definitions of helper functions (name, description, parameter schema compatible with gRPC/OpenAPI, version, permissions required) will be stored in a dedicated SQL database table.
        * This registry allows for dynamic discovery by SERA and enables AI-assisted evolution of helper functions.
        * The RAF API will provide endpoints to query this registry.
    * **Initial Clients (Phase 1):** `NetboxClient`, `CloudstackClient` (using cmk for CloudStack interaction, pending further API evaluation).
    * **Planned Clients:** `ZabbixClient`, `SSHClient`, `RedfishClient`, `ActiveDirectoryClient` (DNS focus), `SaltClient` (for managing Salt configurations, not direct execution).
* **Service Layer** (`services` directory):
    * **ConfigurationService:** Loads and provides access to RAF configurations (API keys, endpoints, default settings) from `config.yaml` (in `/ai/raf/config/`) and environment variables. Supports layered configuration for local dev (secrets in file) vs. future deployments.
    * **LoggingService:** Centralized logging setup, routing logs to `stdout`/`stderr` and to structured log files in `/ai/assets/logs/raf/` (with subdirectories like `reporting`, `automation`).
    * **SQLCacheService:** Provides persistent, queryable caching for data fetched by client libraries. Uses an SQL database. Data can be stored in `/ai/assets/cache/raf/` for preprocessing or large cache items if not suitable for DB.
    * **TaskManagementService:** (Future) Manages asynchronous tasks. Could leverage local L4 GPU (Ollama) for small background AI tasks if applicable.
    * **HelperRegistryService:** Manages the CRUD operations and querying for the Helper Function Registry in the SQL database.
* **Automation Engine** (`automation` directory):
    * Contains logic for executing complex, multi-step automation workflows.
    * **Workflow Definitions:**
        * **YAML:** For simpler, Ansible-like playbooks. Stored in `automation/playbooks/`.
        * **Python Scripts:** For more complex logic. Stored in `automation/workflows/` (e.g., `vm_provisioning.py`).
    * Reusable automation functions can also be placed in `automation/functions/`.
    * Leverages client libraries and services. Exposed via the RAF API.
* **Reporting Engine** (`reporting` directory):
    * Generates reports by fetching data from various client libraries and processing it.
    * **BaseReport Class:** An abstract base class (`reporting/base_report.py`).
    * Individual reports will be modules in `reporting/reports/`.
    * **Data Caching:**
        * Leverages `SQLCacheService` for input data.
        * Reports can specify cache validity or trigger cache refresh.
        * Allows multiple reports to use the same pre-cached data, beneficial for reports scheduled concurrently or for rapid testing/iteration by SERA.
* **CLI Interface** (`raf.py` at `/ai/raf/`):
    * Main entry point for the application, including CLI.
    * Will use a library like Typer or Click.
    * Acts as a client to the RAF API (gRPC or REST).
    * Useful for scripting, manual operations, and diagnostics.

### 3.3. Interaction with SERA Orchestrator

* **Primary Channel:** gRPC API. SERA will be a gRPC client.
* **Discoverability:** SERA will use RAF's gRPC service definitions (Protobuf), the Helper Function Registry API, and LLM Instruction Guides (see Section 6.2) to understand available capabilities and how to use them.
* **Task Execution:** SERA will request RAF to execute specific helper functions, automation workflows, or generate reports by calling the relevant gRPC methods.
* **Data Exchange:** Data will be exchanged via Protobuf messages.
* **Deployment:** Co-location in `/ai/raf/` simplifies initial network configuration.

## 4. Proposed Directory & File Structure

/ai/raf/
├── raf.py                      # Main application entry point (includes CLI)
├── bootstrap_helper.py         # For venv setup
├── requirements.txt
├── alembic.ini                 # Alembic config for DB migrations
│
├── api/                        # API layer (gRPC services, optional FastAPI)
│   ├── __init__.py
│   ├── grpc_server.py          # gRPC server setup
│   ├── rest_main.py            # (Optional) FastAPI app instantiation
│   ├── dependencies.py         # Common dependencies
│   └── services/               # gRPC service implementations (mirroring modules)
│       ├── __init__.py
│       ├── clients_service.py
│       ├── automation_service.py
│       ├── reporting_service.py
│       └── admin_service.py
│   └── protos/                 # Protobuf definitions
│       └── raf.proto
│
├── core/                       # Core engine, workflow logic
│   ├── __init__.py
│   └── engine.py
│
├── clients/                    # Client libraries for external systems
│   ├── __init__.py
│   ├── base_client.py
│   ├── netbox_client.py
│   ├── cloudstack_client.py
│   ├── llm-client-usage-guide.md # LLM guide for general client patterns
│   └── schemas/                # Pydantic models (if REST used) / internal data classes
│
├── services/                   # Internal RAF services (business logic)
│   ├── __init__.py
│   ├── config_service.py
│   ├── logging_service.py
│   ├── cache_service.py
│   └── registry_service.py
│
├── automation/                 # Automation engine
│   ├── __init__.py
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
│   ├── __init__.py
│   ├── llm-reporting-guide.md  # LLM guide for using the reporting engine
│   ├── base_report.py
│   ├── report_orchestrator.py
│   └── reports/
│       └── nb_cs_vm_discrepancy.py
│
├── models/                     # SQLAlchemy models for internal DB
│   ├── __init__.py
│   ├── cache_models.py
│   └── registry_models.py
│
├── config/                     # Default configurations and user config storage
│   ├── __init__.py
│   ├── config.default.yaml
│   └── config.yaml             # User's actual configuration (loaded by config_service)
│
└── utils/                      # Common utility functions
    ├── __init__.py
    └── security.py
│
├── alembic/                    # Alembic migration scripts directory
│   └── versions/
│       └── ...
│
├── tests/                      # Unit and integration tests
│   └── ...
│
└── docs/                       # Documentation
    └── raf_design.md           # This file

Note on `bootstrap_helper.py`: `raf.py` will include the standard bootstrap check:

```python
# In raf.py
import sys

# --- Bootstrap Check ---
# Assuming bootstrap_helper.py is in the same directory as raf.py
try:
    import bootstrap_helper as bootstrap_helper # Ensure bootstrap_helper.py is in /ai/raf/
    # Simplified check, debug flag can be passed differently if needed by Typer/Click
    bootstrap_helper.bootstrap_and_reexecute_if_needed(debug_flag=("--debug" in sys.argv))
except ImportError:
    print("[WARN] bootstrap_helper.py not found. Ensure it is in /ai/raf/.", file=sys.stderr)
    # Potentially exit or handle gracefully
    pass
except Exception as boot_e:
    print(f"[ERROR] Bootstrap failed: {boot_e}", file=sys.stderr)
    # Potentially exit
    pass
# --- Bootstrap Done ---
```

## 5. Data Management & Configuration

* **Configuration** (`/ai/raf/config/config.yaml`, `/ai/raf/config/config.default.yaml`):
    * API keys, external service URLs, database connection strings, default settings. SSH keys for dev can be stored here or path referenced.
    * Loaded by `services/config_service.py`.
* **Internal SQL Database (PostgreSQL):**
    * Will use the existing PostgreSQL instance (sera_ai_dev_postgres) on the development server. A new database within this instance (e.g., `raf_db`) can be created.
    * Managed by SQLAlchemy and Alembic for migrations. Models defined in `/ai/raf/models/`.
    * Stores: Helper Function Registry, SQL Cache metadata, (Future) Task states, Audit logs (if not in separate files).
* **Asset Storage** (`/ai/assets/`):
    * `/ai/assets/logs/raf/`: For RAF-specific logs (application, reporting, automation).
    * `/ai/assets/logs/raf/audit/`: For dedicated audit trail log files.
    * `/ai/assets/cache/raf/`: For file-based caching or pre-processing data if not suitable for the SQL cache.
    * `/ai/assets/tmp/raf/`: For temporary files generated by RAF.

## 6. AI Interaction & Discoverability

### 6.1. Formal Definitions

* **gRPC Service Definitions (Protobuf):** SERA (specifically RAA) will use these from `/ai/raf/api/protos/` to understand available RAF services, methods, and message structures. This is the primary contract for programmatic interaction.
* **Helper Function Registry API:** RAF's gRPC API will provide methods for SERA/RAA to query the list of available helper functions, their detailed descriptions, parameter schemas (translatable to OpenAI function schemas if needed by RAA), versions, and required permissions. This allows discovery of atomic capabilities.

### 6.2. LLM Instruction Guides

* **Purpose:** To provide narrative, procedural context to LLMs (within RAA or other AI agents) on how to effectively use and combine RAF's capabilities for common tasks like creating reports or defining automation workflows. These guides complement the formal API/function definitions by explaining the "how-to" and best practices.
* **Format:** Markdown files (e.g., `llm-reporting-guide.md`, `llm-automation-guide.md`).
* **Content Examples:**
    * `llm-reporting-guide.md`: Would explain the `BaseReport` structure, how to fetch data using client helpers, how to use the cache, and how to format/submit the report.
    * `llm-automation-guide.md`: Would describe how to define YAML playbooks or Python workflows, how to chain helper functions, and principles for error handling in automation.
* **Location:** Stored within the relevant RAF module directories (see Section 4, e.g., `/ai/raf/reporting/llm-reporting-guide.md`).
* **Access by RAA:** RAA's `RAFCapabilityManager` can be designed to fetch these guides (either directly from the filesystem if co-located, or via a simple RAF gRPC endpoint that serves their content). The content of these guides would then be provided as context to RAA's `LLMTaskFormulator`.
* **Benefit:** Reduces the need for the LLM to infer complex procedures solely from code or API schemas, making it easier for AI to generate valid and effective RAF artifacts (reports, workflows).

### 6.3. AI-Assisted Code Generation (Facilitated by RAA)

The LLM logic for generating new report scripts or automation snippets will reside within the "RAF Assistant" (RAA) module in the SERA orchestrator.

RAA will use the formal definitions (gRPC, Helper Registry) and the LLM Instruction Guides from RAF as context for its `LLMTaskFormulator`.

RAA will then interact with RAF's API to register or execute the AI-generated code/configurations, leveraging RAF's secure execution environment.

## 7. Security & Permissions Model

* **Authentication:**
    * **gRPC API:** Secure channel (TLS), potentially token-based authentication for SERA's service account.
    * **CLI:** No direct authentication if run by an already authenticated OS user. RAF should capture the OS username (`getpass.getuser()`) for auditing.
* **User Mapping & Context:**
    * RAF needs to be aware of the initiating user for actions, especially for systems like CloudStack where actions are tied to user API keys, or NetBox for journal entries.
    * SERA will pass user context (e.g., originating AD username, relevant CloudStack API key if pre-configured for the user in SERA) to RAF.
    * `services/config_service.py` might store mappings or methods to retrieve user-specific credentials/API keys based on the context provided by SERA.
* **Authorization (RBAC):**
    * Permissions defined for gRPC API methods, helper functions, and automation workflows.
    * Roles assigned to users/service accounts.
    * Permissions checked at the API layer.
* **SSHClient Specifics:**
    * **Level 1 (Default for AI):** AI (via SERA/RAA) can only trigger pre-defined, audited Python scripts stored in `/ai/raf/automation/scripts/`. Input parameters strictly validated. Scripts designed to return structured JSON.
    * **Level 2 (Restricted AI/Privileged User):** AI/User can request execution of specific, whitelisted commands (e.g., `hostname`, `df -h`). Managed via a configurable list in RAF.
    * **Level 3 (Human Supervised/Emergency):** AI proposes an arbitrary command/script. Requires explicit human approval before RAF executes.
    * No direct AI shell access.
* **SaltClient Specifics:**
    * RAF's `SaltClient` will primarily be used to interact with the Salt Master API for tasks like:
        * Querying minion status, grains, pillar data.
        * Managing Salt pillar configurations (e.g., adding a new server during provisioning).
        * Triggering pre-defined, audited Salt states or orchestrations.
    * Direct execution of arbitrary Salt commands by AI will follow similar restrictions as SSH.
* **Audit Logging:** All significant actions logged with user identity, timestamp, details to `/ai/assets/logs/raf/audit/audit.log` (and potentially to the SQL DB).

## 8. Phase 1 Development Plan & Example Use Case

### 8.1. Focus for Phase 1:

* **RAF Core & API Basics:** Setup gRPC server, basic Core Engine, `ConfigurationService`, `LoggingService`.
* **Client Libraries:** Implement `NetboxClient` and `CloudstackClient` with key helper functions. *Existing code from `netbox_compare.py`, `netbox_cs.py`, and `ncs_app` will be used as a primary source for developing these clients.*
* **Helper Function Registry:** Basic SQL schema and `RegistryService`. gRPC methods to list/describe functions.
* **SQL Cache Service:** Basic implementation.
* **Reporting Engine (Basic):** `BaseReport`, `ReportOrchestrator`. Create an initial `llm-reporting-guide.md`.
* **CLI (`raf.py`):** Basic commands.
* **Internal DB:** Use existing PostgreSQL instance.

### 8.2. Example Use Case: "VM Discrepancy Report (NetBox vs. CloudStack)"

1.  **Define Helper Functions** (with static filters embedded or configurable):
    * `clients.netbox_client.get_filtered_vms(site=None, status=None, fields=[...])` (internally applies `NETBOX_VM_FILTERS`).
    * `clients.cloudstack_client.list_virtual_machines(zone=None, state=None, details=[...])`.
    * Register these in the Helper Function Registry.
2.  **Create Report Module** (`reporting/reports/nb_cs_vm_discrepancy.py`):
    * Inherits from `reporting.base_report.BaseReport`. Uses RAF's gRPC client stubs (or internal calls) to invoke helper functions. Compares data. Formats output.
3.  **Expose via API & CLI:**
    * gRPC method like `GenerateReport(report_name="nb_cs_vm_discrepancy", params={...})`.
    * CLI: `python raf.py report generate nb_cs_vm_discrepancy --output-format json --email-to admin@example.com`.
    * CLI support for grouped reports: `python raf.py report generate daily_vm_summary_group --email-to team@example.com --combine-emails`. (This group would be defined in RAF's config, listing individual reports to run).
4.  **Test with RAA (in SERA):**
    * User prompt to SERA: "Generate the NetBox vs CloudStack VM discrepancy report and email it to me."
    * SERA's RAA module uses its `LLMTaskFormulator`, providing it with the user request, the content of `/ai/raf/reporting/llm-reporting-guide.md`, and relevant helper function schemas from RAF.
    * RAA formulates the call to RAF's `GenerateReport` gRPC method.

## 9. Open Questions & Discussion Points (Updated)

* **API Design Deep Dive:** gRPC decided for SERA-RAF. A REST API via FastAPI can still be an additional, secondary interface if needed for other consumers or simpler CLI interaction, but gRPC is primary.
* **Helper Function Schema:** Will evolve. Needs to be defined in Protobuf for gRPC and translatable to OpenAI schema if SERA's internal "tool" definitions require it.
* **SSH/Salt Command Execution Strategy:**
    * Pre-defined scripts in `/ai/raf/automation/scripts/` (version controlled).
    * AI-generated scripts in `/ai/raf/automation/ai_scripts/` (sandboxed, require review before promotion).
    * Whitelisted commands managed via RAF's configuration.
* **LLM Instruction Guide Content & Access:** What's the best way for RAA to access these? Direct file read (if co-located and permissions allow) or via a simple RAF gRPC endpoint? (Endpoint is cleaner for future separation).
* **Task Management for Long-Running Operations:** Defer detailed design. Synchronous for Phase 1.
* **Configuration Security:** Secrets for local dev in `/ai/raf/config/config.yaml` (appropriately permissioned). The RAF application itself is not planned for K8s initially; the "RAF Assistant" module in SERA (which might be in K8s) will be the secure bridge.
* **Initial Database Choice:** Use the existing local PostgreSQL instance.