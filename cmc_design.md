# CMD Connector (CMC) - Design Document

**Version:** 1.1.0
**Date:** 2026-02-21
**Author(s):** Martin Kecka, Gemini AI
**Category:** Connector (C)
**Acronym:** CMC

---

## 1. Introduction & Purpose

The **CMD Connector (CMC)** is a deterministic and robust Python service designed to be the authoritative gateway for interacting with the **CMD GraphQL API**. Its core purpose is to provide downstream consuming systems (like the Cash Application Assistant or other automation tools) with reliable, formatted access to core client, contact, ERP, and banking master data.

The CMC focuses on three primary functions:

1.  **GraphQL Client Expert:** Encapsulate complex GraphQL query and authentication logic, providing simple, stable Python functions (e.g., `get_client_by_email`, `validate_invoices`). It handles rate limiting, authentication, and error handling transparently.
2.  **Schema Discovery & Exploration:** Provide developer tools (CLI) to automatically pull the current GraphQL schema and generate filtered, readable summaries of core data types (`client_`, `billing_`, `person`).
3.  **Data Provider (gRPC Server):** Serve both live GraphQL API results and cached/processed local data assets (like invoice rules and bank maps) via a high-performance **gRPC API**, ensuring data is readily available without direct API calls to CMD.

---

## 2. Architecture & Core Components 🧱

The CMC is a standalone Python application that exposes both a Command Line Interface (CLI) for development and maintenance, and a gRPC service for runtime data access.

### Core Components

* `cmd_connector.py`: The application entry point that handles configuration loading, logging setup, and launching the `click` CLI.
* `cli/cmc_cli.py`: Defines all external CLI commands (e.g., `start-grpc-server`, `discover-schema`, `refresh-local-data`, `list-invoices`).
* `core/cmd_client.py`: **The API Core**. Manages OAuth2 token acquisition and executes all parameterized GraphQL queries, handling retries and rate limiting via the `tenacity` library.
* `core/graphql/`: Stores all predefined GraphQL queries (`.gql` files) used by the `CmdClient`.
* `core/discovery.py`: Implements the GraphQL introspection query for schema discovery.
* `core/targeted_explorer.py`: Filters and summarizes the discovered schema for relevant core business entities, removing extraneous noise types (e.g., aggregation/input types).
* `core/schema_explorer.py`: Provides deep-dive inspection capabilities into specific GraphQL types and their fields.
* `core/query_lister.py`: Utility to parse the discovered schema and list all available top-level queries and their expected arguments.
* `core/rule_generator.py`: **The Asset Generator**. Responsible for connecting to the live CMD API, fetching raw ERP and banking data, and generating the optimized local asset files (Invoice Rules, Bank Maps, and Validation files).
* `grpc_api/`: The external communication layer.
    * `grpc_api/protos/cmc_service.proto`: Defines the gRPC service and all data transfer objects (DTOs).
    * `grpc_api/generate_stubs.py`: A pure Python utility executed at server startup to automatically compile `.proto` files into up-to-date Python stubs, eliminating the need for shell scripts.
    * `grpc_api/services.py`: Implements the `CmdConnectorServiceServicer`, providing logic to fulfill RPC requests from either the `CmdClient` (for live API calls) or local assets (for static data).
* `utils/`: Core application utilities.
    * `utils/logging_setup.py`: Standardizes application-wide logging formats and rotation policies.
    * `utils/rpc_transaction_logger.py`: Intercepts and logs all gRPC request payloads and responses to local JSON files for comprehensive auditing and debugging.
* `assets/`: Storage for processed data derived from the API for fast local access.
    * `invoice_rules.json`: Generated document parsing rules.
    * `bank_currency_map.json`: Generated map linking bank accounts to available currencies.
    * `live_open_invoices.csv`: The raw dump of open ERP transactions.
    * `bu_bank_info_complete_validation.csv`: Generated complete banking info file enriched with secondary currencies for validation.

---

## 3. Key Features & Functionality

### 3.1. gRPC API Interface

The gRPC service is the primary method of runtime consumption. It offers two categories of methods:

| Category | Method | Description | Source |
| :--- | :--- | :--- | :--- |
| **Live API** | `GetClientByEmail` | Finds a client (exact or domain match) by email address. | `cmd_client.py` (GraphQL) |
| **Live API** | `GetClientAllData` | Fetches a deep, comprehensive data dump for a client ID or Name. | `cmd_client.py` (GraphQL) |
| **Live API** | `ValidateInvoices` | Validates if a list of invoice numbers exists as open ERP transactions. | `cmd_client.py` (GraphQL) |
| **Live API** | `GetInvoicesByBan` | Retrieves recent ERP transactions (invoices) for a specific Billing Account Number (BAN). | `cmd_client.py` (GraphQL) |
| **Asset Data** | `GetOpenInvoices` | Retrieves open ERP transactions, with optional date filtering. | `assets/live_open_invoices.csv` |
| **Asset Data** | `GetInvoiceRules` | Retrieves the generated invoice number parsing rules. | `assets/invoice_rules.json` |
| **Asset Data** | `GetBuBankInfo` | Retrieves primary banking info enriched with secondary currencies. | `cmd_client.py` + `assets/bank_currency_map.json` |

### 3.2. Local Asset Generation (`refresh-local-data`)

The `core/rule_generator.py` script performs a multi-step process to create high-value local assets for data applications:

1.  **Fetch Live Data:** Calls the CMD API to get all open ERP transactions (`get_erp_transactions`) and legal entity banking details (`get_legal_entities_banking_info`).
2.  **Generate Invoice Rules:** Analyzes recent ERP transactions to derive and save generic invoice naming patterns (prefix, document length, example number) to `invoice_rules.json`.
3.  **Generate Currency Map:** Analyzes the ERP data to determine which currencies have been remitted to which bank account numbers, creating the `bank_currency_map.json`.
4.  **Generate Combined Bank Info:** Takes the live primary banking details and enriches them with the secondary currencies from the generated currency map, outputting a consolidated validation file (`bu_bank_info_complete_validation.csv`).

### 3.3. GraphQL Resilience

The `CmdClient` is built for reliability:

* **Authentication:** Manages OAuth2 token refresh automatically.
* **Retry Logic:** Uses `tenacity` to automatically retry API calls on common transient errors and implements specific backoff strategies for **HTTP 429 (Rate Limit)** errors.
* **Schema Consistency:** All queries are stored as static `.gql` files to ensure stability and simplify maintenance.

---

## 4. Data Flows and Configuration

### Data Flow

1.  **Live Requests:** `gRPC Client` $\rightarrow$ `services.py` $\rightarrow$ `CmdClient` $\rightarrow$ **CMD GraphQL API**
2.  **Asset Requests:** `gRPC Client` $\rightarrow$ `services.py` $\rightarrow$ `_read_and_parse_csv`/`_read_and_parse_json` $\rightarrow$ **`assets/` Files**
3.  **Asset Refresh:** `CLI` (`refresh-local-data`) $\rightarrow$ `rule_generator.py` $\rightarrow$ `CmdClient` $\rightarrow$ **CMD GraphQL API** $\rightarrow$ **`assets/` Files**

### Configuration (`config/settings.yaml`)

| Setting Key | Purpose |
| :--- | :--- |
| `grpc_server` | Defines the listener host and port (`0.0.0.0:50053`). |
| `target_system` | Stores all critical CMD API access parameters (URLs, `client_id`, `client_secret`, `scope`, and timeouts). |
| `logging` | Defines log level and enables file logging. |
| `transaction_logging` | **Enables transaction logging** for all gRPC calls, recording payload and response data to disk for auditing/debugging. |
| `paths` | Specifies the location for the source `protos` (`proto_source_dir`) and the required destination path for **generated stubs** (`generated_stubs_path`). |

---

## 5. Command-Line Interface (CLI) Usage

The `cmc_cli.py` module uses `click` to provide core developer commands:

```bash
# Start the gRPC server (primary mode of operation, dynamically generates stubs if needed)
./cmd_connector.py start-grpc-server

# Discover and save the full CMD GraphQL schema
./cmd_connector.py discover-schema

# Filter the full schema to extract high-value entity definitions
./cmd_connector.py inspect-targeted

# Inspect a specific GraphQL type and its fields in detail
./cmd_connector.py inspect-type "billing_erp_transactions"

# List all available top-level queries based on the discovered schema
./cmd_connector.py list-queries

# Fetch the latest ERP transactions and banking data, then generate local asset files
./cmd_connector.py refresh-local-data

# Test the client lookup logic by email
./cmd_connector.py get-client-by-email --email "john.doe@example.com"

# Fetch exhaustive client data by ID or exact Name
./cmd_connector.py get-client --id 12345
./cmd_connector.py get-client --name "Acme Corp"

# Fetch recent ERP transactions/invoices for a specific Billing Account Number (BAN)
./cmd_connector.py list-invoices --ban "T194919" --months 6
```

## 6. Future Enhancements

The CMC's future development path aligns it with the **SERA Smart Connector Pattern** (Category C), transforming it from a simple data pipe into a fully discoverable and intelligent asset for the entire ecosystem.

### 6.1. Formalize AI-Centric Discoverability

The CMC must formally register its capabilities with the Orchestrator, ensuring all **Assistants** and **Processors** can use it dynamically.

* **Tool Definition Contract:** Create a `tools.d/cmd_connector.yaml` file in the Orchestrator's config. This file will map the gRPC methods (e.g., `GetClientByEmail`, `ValidateInvoices`) to clear, simple, LLM-readable descriptions and input schemas, adhering to the SERA standard (Section 10.2).
* **Dynamic Stub Placement:** Ensure the build process places the compiled gRPC stubs (`cmc_service_pb2.py`, etc.) into the shared asset path (`/ai/assets/protos/cmd_connector/generated`), enabling the Orchestrator to load them dynamically at runtime without static dependencies.
* **LLM Instruction Guide:** Create a detailed `llm_instruction_guide.md` to instruct the Orchestrator's reasoning model on the nuances of using CMC functions (e.g., how to interpret domain matches or when to fall back from a live call to a local asset).

### 6.2. Implement Data Analysis Utility for Efficiency (The "Panda" Tool)

To prevent the LLM from being overwhelmed by large datasets returned by queries (a major architectural challenge, as detailed in SERA Section 6.2), the CMC should include a simple internal analytical tool.

* **Internal `AnalyzeData` Method:** Implement a private utility method in `services.py` that can load the data from `GetOpenInvoices` into a `pandas.DataFrame`. This method would accept structured, AI-generated parameters (e.g., `filter_by: { "business_unit": "US100BU" }`, `aggregate: "sum(balance)"`).
* **Expose Simple RPC:** Expose a minimal RPC (e.g., `GetAggregatedInvoices`) that calls the internal analysis utility, allowing the Orchestrator to retrieve a small, pre-processed summary (e.g., total open balance for a BU) instead of the entire dataset.

### 6.3. Enforce User Identity Auditing

Adhering to the SERA **Header-Based RBAC** standard (Section 6.6), the CMC must record the actual end-user who initiated the request.

* **Inbound Header Logging:** Update the gRPC service implementation in `grpc_api/services.py` to read the `X-Forwarded-User` header (passed from the Orchestrator via gRPC context) and include it in the **Transaction Logging** entry. This ensures the full audit trail links every executed CMD API call back to the authenticated SERA user.

### 6.4. Enhance Client Lookup Resilience

* **Contextual Client Disambiguation:** The current `get_client_by_email` logic returns multiple clients on a `DOMAIN_MATCH`. The future enhancement is to pass the resulting list and the original `payer_name` from an external source (like a BSI transaction) to the Orchestrator. The Orchestrator's reasoning LLM can then use the `gemma3:27b` model to perform a fuzzy name comparison against the candidate clients, making an **intelligent decision** on the best match before returning the final client ID.