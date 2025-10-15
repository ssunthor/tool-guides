# /ai/assets/docs/tool_guides/graylog_suite.md
# Graylog Suite: Design & Implementation Plan (v2.0)

## 1. Introduction & Strategic Goal 🎯
This document outlines the design and phased implementation plan for the Graylog Suite, a new module for interacting with the Graylog log management platform. The design strictly adheres to the established SERA Ecosystem Architecture, separating concerns into two distinct components:

* **The Graylog Connector (GLC):** A deterministic, non-AI (C) Connector. Its mission is to provide a robust, stable, and easy-to-use gRPC interface that acts as a wrapper around the Graylog REST API. It handles the low-level details of API communication and query execution.
* **The Graylog Assistant (GLA):** An AI-driven (A) Assistant and (P) Processor. Its mission is to serve as an intelligent analyst. It translates high-level, natural language questions into formal Graylog queries, executes them via the GLC, analyzes the results, and provides human-readable summaries.

This two-module approach ensures that the components are modular, independently testable, and aligned with the core principles of grounding and specialization defined in the SERA architecture.

---

## 2. Module Architecture & Responsibilities
### 2.1. The Graylog Connector (GLC) - The "Executor"
The GLC is the resilient execution layer. It is a backend gRPC service that takes structured, deterministic commands and translates them into Graylog REST API calls.

**Key Responsibilities:**

* **gRPC Service Interface:** Exposes a strongly-typed gRPC API for all interactions. This is the single entry point for the GLA.
* **Graylog API Abstraction:** Manages all aspects of communicating with the Graylog REST API, including authentication, session management, and endpoint specifics.
* **Structured Query Execution:** Accepts a structured `SearchRequest` and is solely responsible for building the correct Lucene-based query string and time range parameters.
* **Stream & Alert Management:** Provides simple gRPC methods for future CRUD (Create, Read, Update, Delete) operations on Graylog streams and alerts.
* **Health & Status:** Exposes a gRPC health check endpoint (`grpc.health.v1.Health`).
* **Future-Proof Transformation Engine:** While initial search functionality is a simple translation, the GLC is architected to include a Pre-Flight Transformation Engine for future "write" operations. This engine will handle validation and name-to-ID resolution for creating and modifying streams or alerts, mirroring the NBC's robust design.
* **Zero AI:** This module contains no LLM logic. Its behavior is entirely programmatic and predictable.

### 2.2. The Graylog Assistant (GLA) - The "Analyst"
The GLA is the cognitive layer and the user-facing component. It leverages a Large Language Model to bridge the gap between human questions and machine data.

**Key Responsibilities:**

* **User Interface:** Provides a standalone Streamlit UI. The UI will feature expandable sections for transparency, allowing users to inspect the generated gRPC requests and the raw data returned at each step.
* **AI-Powered Analysis Pipeline:** Implements a sophisticated, multi-stage pipeline for handling user requests:
    * **Intent & Entity Recognition:** An LLM call translates a user's question (e.g., "Show me all SSH login failures from yesterday") into a structured representation of their goal.
    * **Structured Query Generation:** A second LLM call takes these entities and generates the formal `SearchRequest` message needed by the GLC.
    * **Dry Run & User Confirmation:** For any requests that will modify Graylog (e.g., creating a stream), the GLA will first perform a dry run, generate a human-readable summary of the intended changes, and require explicit user confirmation before proceeding.
    * **Execution via gRPC:** Calls the GLC's gRPC methods with the generated request.
    * **AI-Powered Failure Analysis:** If a gRPC call fails, the GLA uses an LLM to analyze the error message in context and attempt to generate a corrected request for an automatic retry, enabling self-correction.
    * **Deterministic Data Analysis:** For queries requiring aggregation (e.g., count, top 5), the GLA receives raw data from the GLC and uses a dedicated data processing engine (e.g., using the Pandas library) to perform the calculations reliably.
    * **Final Answer Synthesis:** The final, pre-processed data is sent to an LLM to generate a concise, human-readable summary and answer.

---

## 3. gRPC API Definition (glc_service.proto)
The communication contract between the GLA and GLC is defined by this Protocol Buffers file.

```proto
syntax = "proto3";

package graylog_connector.v1;

import "google/protobuf/struct.proto";
import "google/protobuf/timestamp.proto";

// The main service definition for the Graylog Connector (GLC)
service GraylogConnectorService {
  // Executes a search against Graylog and returns the results.
  rpc ExecuteSearch(SearchRequest) returns (SearchResponse) {}

  // Future methods for managing streams and alerts. These will leverage
  // the GLC's Pre-Flight Transformation Engine.
  // rpc GetStream(GetStreamRequest) returns (Stream);
  // rpc CreateStream(CreateStreamRequest) returns (Stream);
  // rpc DeleteAlert(DeleteAlertRequest) returns (DeleteAlertResponse);
}

// Represents a single filter condition for a search.
message Filter {
  string field = 1; // The log field to filter on (e.g., "source", "level").
  string value = 2; // The value to search for.
  enum Operator {
    EQUAL = 0;
    NOT_EQUAL = 1;
    CONTAINS = 2; // Maps to a simple query string search.
    EXISTS = 3;   // Checks for the existence of a field.
  }
  Operator operator = 3;
}

// The request message for the ExecuteSearch RPC.
message SearchRequest {
  enum LogicOperator {
    AND = 0;
    OR = 1;
  }
  LogicOperator logic_operator = 1;
  repeated Filter filters = 2;

  google.protobuf.Timestamp start_time = 3;
  google.protobuf.Timestamp end_time = 4;

  int32 limit = 5; // The maximum number of log messages to return.
}

// A single log message returned from Graylog.
message LogMessage {
  string id = 1;
  google.protobuf.Timestamp timestamp = 2;
  string source = 3;
  string message = 4;
  google.protobuf.Struct all_fields = 5;
}

// The response message for the ExecuteSearch RPC.
message SearchResponse {
  bool success = 1;
  string error_message = 2;
  repeated LogMessage messages = 3;
  int32 total_results = 4;
}
```

---

## 4. Detailed Workflow Example
This example traces a user request through the entire updated suite.

**User Request:** "What are the top 5 sources of 404 errors on our web servers in the last 6 hours?"

**Step 1: GLA - Intent & Query Generation**

* The user enters the query into the GLA's Streamlit UI.
* **LLM Call #1 (Intent/Entities):** Extracts `{ "http_response": 404, "service": "web-servers", "time_range": "last 6 hours", "group_by": "source", "limit": 5 }`.
* **LLM Call #2 (Plan Generation):** The GLA's planner generates the gRPC `SearchRequest` message. To ensure enough data for analysis, it uses a high retrieval limit.
    * **Filters:** `[ { field: "http_response", value: "404", operator: EQUAL }, { field: "application_name", value: "web-servers", operator: CONTAINS } ]`
    * **Time Range:** Calculates timestamps for the "last 6 hours".
    * **Limit:** 2000 (a high limit for raw data retrieval).

**Step 2: GLA-to-GLC - gRPC Execution**

* The GLA (client) sends the `SearchRequest` to the GLC's (server) `ExecuteSearch` method.

**Step 3: GLC - Execution**

* The GLC receives the `SearchRequest`.
* It programmatically constructs the final Lucene query string: `http_response:404 AND application_name:*web-servers*`.
* It makes the REST API call to Graylog and returns the `SearchResponse` containing up to 2000 raw log messages.

**Step 4: GLA - Deterministic Data Analysis**

* The GLA receives the `SearchResponse`. It does not send the raw logs to the LLM.
* It loads the messages into a Pandas DataFrame.
* It executes a deterministic operation to find the top 5 sources: `df.groupby('source')['source'].count().nlargest(5)`.
* The result is a small, aggregated data structure, like `{'192.168.1.101': 152, '10.0.5.23': 98, ...}`.

**Step 5: GLA - Final Answer Synthesis**

* **LLM Call #3 (Synthesis):** The GLA sends the original user question and the small, aggregated data structure to the final synthesis prompt.
* The LLM generates the final, human-readable answer.
* The GLA displays the answer in the Streamlit UI:
    ```
    Here are the top 5 sources of 404 errors in the last 6 hours:
    1. 192.168.1.101 (152 errors)
    2. 10.0.5.23 (98 errors)
    ...etc.
    ```

---

## 5. Phased Implementation Plan

### Phase 1: Build the Graylog Connector (GLC)
**Goal:** Create the deterministic, non-AI gRPC service.

1.1: **Project Setup:** Standard Python project setup (copy bootstrap_helper.py and utils from NBC, create similiar folder structure).
1.2: **Define gRPC API:** Create `glc_service.proto` and generate stubs.
1.3: **Implement Core Graylog Client:** Create `GraylogClient` class. Design placeholder logic for the future Pre-Flight Transformation Engine.
1.4: **Implement gRPC Service:** Implement the `ExecuteSearch` method with robust error handling.
1.5: **Create Server Entrypoint:** Create the main `graylog_connector.py` server script.
1.6: **Test Rigorously:** Use a test client to validate `SearchRequest` functionality (similiar like in NBC tests.py).

### Phase 2: Build the Graylog Assistant (GLA)
**Goal:** Create the AI-driven Streamlit UI and analysis engine.

2.1: **Project Setup:** Standard Python project setup.
2.2: **Implement gRPC Client:** Create a client class to communicate with the GLC.
2.3: **Develop Prompts:** Create prompts for `query_generation`, `synthesis`, and the new `failure_analysis`.
2.4: **Build Core Pipeline & UI:**
    * In `ui/gla_app.py`, create the main Streamlit application.
    * Implement the full multi-stage analysis pipeline function.
    * Build the deterministic data analysis module using Pandas.
    * Implement the dry run confirmation logic for future write operations.
    * Add expandable sections to the UI for transparency.
2.5: **Create Launcher:** Create `graylog_assistant.py` to launch the app.

### Phase 3: Auditing & Hardening
**Goal:** Implement production-grade logging and auditing.

3.1: **Implement Transaction Logging:** Add the `PipelineTransactionLogger` to both the GLC and GLA to create a complete, auditable trail for every request, mirroring the NBA/NBC standard.

---

## 6. Configuration (settings.yaml)
#### `/ai/graylog_connector/config/settings.yaml`
```yaml
#
# Main configuration file for the Graylog Connector (GLC) module.
#

grpc_server:
  host: "0.0.0.0"
  port: 50082

target_system:
  api_url: "https://graylog-prod.gt-t.net/api"
  # Use a base64-encoded token for authentication (user:token)
  api_token: "bXNlcmE6Y2FjNzUtYTJjMy00YjcwLTg0MjEtZDE4OTMyYjEyM2Fi"
  timeout: 30

logging:
  level: "INFO"
  directory: "/ai/assets/logs/glc_connector"
  enable_file_logging: true
  app_log_filename: "glc.log"
  error_log_filename: "glc_err.log"

transaction_logging:
  enabled: true
  # Directory is relative to the main logging directory
  directory: "transactions"
  retain_count: 50

paths:
  # Path to the shared assets directory, relative to the module root.
  shared_assets_path: "../assets"
  # Directory containing the source .proto files, relative to the module root.
  proto_source_dir: "grpc_api/protos"
  # Absolute path where the generated Python gRPC stubs will be placed.
  generated_stubs_path: "/ai/assets/protos/graylog_connector/generated"
```

#### `/ai/graylog_assistant/config/settings.yaml`
```yaml

#
# Main configuration file for the Graylog Assistant (GLA) module.
#

# --- Connection to the Graylog Connector (GLC) gRPC Service ---
glc_service_config:
  host: "localhost"
  port: 50082
  timeout_seconds: 60

# --- Connection to the LLM (Ollama) ---
llm_interface:
  ollama_client:
    base_url: "https://aif-llm-ollama-gemma3-27b.slo.gt-t.net/"
    model: "gemma3:27b"
    chat_timeout_seconds: 180
    ssl_verify: true
    num_ctx: 32768

# --- Logging Configuration ---
logging:
  level: "INFO"
  directory: "/ai/assets/logs/gla_assistant"
  enable_file_logging: true
  app_log_filename: "gla.log"
  error_log_filename: "gla_err.log"

transaction_logging:
  enabled: true
  # Directory is relative to the main logging directory
  directory: "transactions"
  retain_count: 50

# --- File paths relative to the module root ---
paths:
  # The main Streamlit application script
  ui_script_path: "ui/gla_app.py"
  # Path to the shared assets directory, used to find stubs
  shared_assets_path: "../assets"
  # Path where the assistant will look for the generated gRPC stubs
  generated_stubs_path: "../assets/protos/graylog_connector/generated"

# --- Default User Context for CLI/UI testing ---
user_context:
  username: "local_user"
  display_name: "Local User"
```

---

## 7. Logging & Auditing
The suite will implement a robust, two-tiered logging strategy for observability and auditing.

* **Application Logging:** Standard service lifecycle events (startup, health checks, internal errors) will be logged to `gla.log`/`glc.log` for monitoring the health of the services themselves.
* **Transaction Logging:** This is the core auditing feature. When enabled, the `PipelineTransactionLogger` in each module creates a unique, timestamped JSON file for every single transaction. This log captures the entire lifecycle of a request, including the initial query, all AI-generated plans, the exact gRPC payloads, the results or errors, and the final synthesized answer. This provides an exhaustive, step-by-step audit trail for every action the system takes.