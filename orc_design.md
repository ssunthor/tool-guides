# Oracle Connector (ORC) - Design Document

**Version:** 1.1.0
**Date:** 2026-02-21
**Author:** Martin Kecka & Gemini AI
**Category:** Connector (C)
**Acronym:** ORC

---

## 1. Introduction & Purpose

The **Oracle Connector (ORC)** is a specialized, hybrid integration service designed to bridge the SERA ecosystem with **Oracle Cloud ERP**. Because Oracle's interface landscape is complex, the ORC acts as an abstraction layer that intelligently routes requests between **REST APIs** (for high-speed transactional data) and **SOAP Web Services** (for heavy reporting and complex queries).

The ORC focuses on three primary functions:

1.  **Hybrid Protocol Gateway:** Seamlessly manages the dichotomy of Oracle ERP. It uses REST for standard entities (Invoices, Receipts) and SOAP/XML for BI Publisher reports (Bank Statements, Mass Data), handling the specific authentication nuances (WS-Security) transparently.
2.  **Intelligent Data Parsing:** transforming raw, verbose Oracle XML and MT940 reports into clean, structured Python dictionaries and Protocol Buffers usable by AI agents.
3.  **Dynamic Discovery:** Includes a robust "Inspector" module that crawls Oracle's massive API surface (WSDLs and REST Catalogs) to generate local documentation and schema definitions, aiding developer understanding of the ERP.

---

## 2. Architecture & Core Components 🧱

The ORC is a Python application exposing a `click` CLI for management/discovery and a `gRPC` server for runtime operations.

### Core Components

* `oracle_connector.py`: The application entry point (bootstrapping, logging, CLI launch).
* `cli/orc_cli.py`: The operator interface. Handles server management, manual report execution, and the interactive `discover` commands.
* **Communication Layer:**
    * `core/rest_client.py`: Handles JSON interactions. Features custom pagination loops ("Search Light, Fetch Heavy") to overcome Oracle REST filtering limitations.
    * `core/soap_client.py`: Wraps `zeep` to handle BI Publisher and AR Service interactions. Implements the custom `UsernameTokenWithTimestamp` required by Oracle's WS-Security.
* **Data Processing & Utilities:**
    * `core/parsers/oracle_xml_parser.py`: Uses `xml.etree` and `pandas` to flatten complex BI Publisher XML hierarchies into linear datasets.
    * `core/parsers/mt940_parser.py`: (In Development) specialized Regex parser for extracting Swift MT940 bank statement tags (`:61:`, `:86:`) from text blobs.
    * `core/report_processors.py`: Post-processes Master Data reports (e.g., filtering out inactive BUs, dummy banks, and POBO accounts from bank account mapping).
    * `utils/logging_setup.py`: Manages root logging, rotating file handlers for app and error logs.
    * `utils/rpc_transaction_logger.py`: Automatically logs all inbound gRPC requests and outbound responses to timestamped JSON files for audit trails.
* **Discovery Module:**
    * `core/rest_inspector.py`: Downloads and indexes Oracle REST API catalogs (`/describe`).
    * `core/wsdl_inspector.py`: Introspects SOAP WSDLs to generate human-readable summaries of available services and types.
* **gRPC Layer:**
    * `grpc_api/protos/orc.proto`: Defines the strict contract for interactions (e.g., `FindReceiptByInvoice`, `ApplyCreditMemo`).
    * `grpc_api/generate_stubs.py`: Automates the compilation of protobuf stubs (`orc_pb2.py`, `orc_pb2_grpc.py`) directly before server initialization to ensure schema sync.
    * `grpc_api/service.py`: The servicer that orchestrates calls between the gRPC request, the appropriate Client (REST vs. SOAP), and the Parsers.

---

## 3. Key Features & Functionality

### 3.1. gRPC API Interface

The gRPC service abstracts the underlying protocol (REST vs SOAP) from the consumer.

| Method | Protocol | Description | Logic Pattern |
| :--- | :--- | :--- | :--- |
| `GetBankStatements` | **SOAP** | Fetches MT940 bank statements for a date range. | Run BI Report $\rightarrow$ XML Parse $\rightarrow$ MT940 Parse |
| `CreateStandardReceipt` | **Hybrid** | Creates an AR receipt (e.g., for batch payments) and applies it to an invoice. Includes automated Cross-BU IC Bridging. | Direct POST (REST) $\rightarrow$ Apply (SOAP) |
| `CreateUnappliedReceipt`| **REST** | Creates a standalone unapplied AR receipt. | Direct POST |
| `GetStandardReceipts` | **REST** | Fetches detailed data for a list of Receipt Numbers or a specific ID. | Direct GET (Loop) |
| `FindReceiptByInvoice` | **REST** | Finds which receipt covers a specific invoice. | **Pagination Loop:** Query by BAN (Newest First) $\rightarrow$ Client-side Filter |
| `FindInvoicesByBan` | **REST** | Retrieves open invoices for a specific Customer Account (BAN). | Query with finder (`invoiceSearch`) |
| `GetInvoiceDetails` | **REST** | Fetches standard details for a specific invoice number. | Auto-resolve ID $\rightarrow$ Direct GET |
| `GetCreditMemoDetails`| **SOAP/REST**| Retrieves credit memo history and application statuses. | Run BI Report $\rightarrow$ Smart Link Processing |
| `ApplyCreditMemo` | **Hybrid** | Applies a credit memo to a specific invoice balance. | Auto-resolve IDs (REST) $\rightarrow$ Apply (SOAP) |
| `UnapplyCreditMemo` | **SOAP** | Unapplies a specific receivable application ID. | Direct SOAP Call |

### 3.2. "Search Light, Fetch Heavy" Pattern

To circumvent Oracle REST API limitations (specifically the inability to filter parents by child attributes), the ORC implements a specialized logic pattern in `rest_client.py`:
1.  **Query:** Fetch a lightweight list of recent objects filtered only by indexed fields (e.g., `CustomerAccountNumber`), sorted by `CreationDate:desc`.
2.  **Scan:** Iterate through the results client-side, expanding child resources (`remittanceReferences`) where possible.
3.  **Fetch:** Only when a match is found locally is the full, heavy payload fetched for the specific object ID.

### 3.3. Discovery & Introspection

The ORC treats the ERP schema as dynamic. The `discover` CLI group allows developers to map the environment:
* `discover init`: Downloads the latest WSDLs and REST Catalogs from the target environment.
* `discover search <keyword>`: Searches local cached definitions to find API endpoints and attribute types without needing to browse Oracle documentation.
* `discover inspect <resource>`: Shows the exact structure, mandatory fields, and children of a resource.

### 3.4. Cross-BU Inter-Company (IC) Bridge Logic

When executing `CreateStandardReceipt`, the ORC employs a Hybrid flow: creating the receipt header via REST, then applying it via SOAP. If Oracle returns a Cross-BU error (e.g., "receipt business unit can't process payments" or "cross currency"), the ORC automatically orchestrates an IC Bridge:
1. Extracts source and target Business Unit IDs.
2. Performs a SOAP Write-Off on the original receipt against an IC Activity Account.
3. Generates a Child Receipt (`[ReceiptNumber]-IC`) in the target BU via REST.
4. Applies the newly created Child Receipt to the target Invoice via SOAP.

### 3.5. Smart Credit Memo Processing

For `GetCreditMemoDetails`, the ORC combines SOAP reporting with active logic. It parses the BI Publisher report for the CM, groups applications by Invoice ID, calculates the net balance, and tags historical rows with `ORC_LINK_STATUS` (`ACTIVE`, `HISTORY`, `CONTRIBUTING`). If the report is empty, it falls back to a REST query to verify the CM's existence, returning a clean state.

---

## 4. Data Flows and Configuration

### Data Flow: Bank Statement Processing

1.  **Request:** gRPC `GetBankStatements` request received.
2.  **Execution:** `SoapClient` injects WSSE Security Header and calls `PublicReportService.runReport`.
3.  **Retrieval:** Oracle returns a Base64 encoded XML blob.
4.  **Parsing:**
    * `OracleXMLParser` extracts the raw text from the XML wrapper.
    * `MT940Parser` iterates through Swift tags to structure the transaction data.
5.  **Response:** Structured Protobuf response returned to Orchestrator.

### Configuration (`config/settings.yaml`)

| Setting Key | Purpose |
| :--- | :--- |
| `oracle_erp` | Credentials (`username`/`password`) and WSDL endpoints. |
| `paths` | Locations for caching WSDLs (`discovery/`) and gRPC stubs. |
| `grpc_server` | Port configuration (Default: `50054`). |
| `logging` | File rotation settings for `orc_app.log` and `orc_err.log`. |

---

## 5. Command-Line Interface (CLI) Usage

The `orc_cli.py` provides distinct groups for different business domains:

```bash
# Start the server (generates stubs automatically)
./oracle_connector.py start-grpc-server

# Discovery: Initialize local cache of Oracle APIs
./oracle_connector.py discover init --api fscm

# Receipts: Find a receipt by BAN and Invoice (uses the Logic Pattern)
./oracle_connector.py receipts search-by-ref --ban "T174216" --invoice "IN12345"

# Reports: Manually run a BI Publisher report and save XML
./oracle_connector.py reports run "/Custom/Financials/MyReport.xdo" --output test.xml

# Master Data: Fetch and filter active Bank Accounts to local cache
./oracle_connector.py master-data bank-accounts --save

# Invoices: List open invoices for a specific Customer Account (BAN)
./oracle_connector.py invoices list-by-ban --ban "T174216" --full

# Credit Memos: Apply a specific Credit Memo to an Invoice
./oracle_connector.py credit-memos apply --cm "CM-9901" --inv "IN12345" --amount 500.00
```

Here is the updated Section 6 for the orc_design.md file, aligned with the architectural standards and depth of the CMC reference document.

Markdown

## 6. Future Enhancements

The ORC's development roadmap aligns with the **SERA Smart Connector Pattern** (Category C), aiming to transform it from a static protocol gateway into an intelligent, self-describing asset for the AI ecosystem.

### 6.1. Formalize AI-Centric Discoverability

The ORC must formally register its capabilities with the Orchestrator, ensuring that Assistants can dynamically understand how to query Oracle without hard-coded logic paths.

* **Tool Definition Contract:** Create a `tools.d/oracle_connector.yaml` configuration file. This will map the complex gRPC methods (e.g., `FindReceiptByInvoice`) to simplified, LLM-readable descriptions and input schemas.
* **Dynamic Schema Exposure:** The current `discover` CLI module serves human developers. A future enhancement will expose a `GetResourceDefinition` RPC. This allows an LLM to query the connector (e.g., *"What are the required fields for a Standard Receipt?"*), enabling the AI to self-correct payload errors during runtime.

### 6.2. Enforce User Identity Auditing

Adhering to the SERA **Header-Based RBAC** standard (Section 6.6), the ORC must ensure that actions taken in Oracle Cloud are traceable to the specific human or agent initiating them.

* **Inbound Header Logging:** Update `grpc_api/service.py` to capture the `X-Forwarded-User` header from the gRPC context.
* **Transaction Context:** While the technical connection to Oracle uses a service account (`Sera-AI`), the internal transaction logs (`rpc_transaction_logger`) must record the `X-Forwarded-User`. This creates a bridge between the generic Oracle user and the specific SERA operator for audit purposes.

### 6.3. Transaction Replay & Self-Healing

Given the complexity of the "Search Light, Fetch Heavy" pattern and the potential for transient Oracle REST errors, the system requires a mechanism to recover from partial failures.

* **Replay RPC:** Implement a `ReplayTransaction` RPC that accepts a Transaction ID from the logs. It will reconstruct the exact payload and logic path used in a failed attempt, allowing for "Safe Retry" without user re-entry.
* **Adaptive Pagination:** Enhance the `find_receipt_by_invoice_loop` to be "adaptive." If O