# [Connector Name] ([Acronym]) - Design Document

**Version:** 1.1
**Date:** [Date]
**Author(s):** [Author Name]
**Category:** Connector (C)
**Acronym:** [Acronym]

---

## 1. Introduction & Purpose

The **[Connector Name] ([Acronym])** is a specialized gRPC microservice within the SERA ecosystem. It serves as the single, authoritative, and deterministic gateway for all interactions with the **[Target System Name]** API.

It acts as an expert tool for higher-level (A) Assistants. Its purpose is to solve the fragility of AI-generated API payloads by accepting simplified, logical gRPC requests and translating them into syntactically perfect REST API calls that the target system is guaranteed to understand. This separation of concerns creates a more resilient and maintainable automation architecture.

---

## 2. Module Category & Acronym

* **Category:** Connector (C)
* **Acronym:** [Acronym]

---

## 3. Architecture & Integration

* **Service Type:** Standalone gRPC Service.
* **Called By:** Any authorized module, typically an (A) Assistant or the Sera Orchestrator.
* **Interacts With:**
    * The [Target System Name] API endpoint.
    * (Optional) Internal schema files it generates/maintains for its transformation logic.

---

## 4. Core Functionality (Recommended Pattern)

The Connector's functionality is best divided into two processes that ensure it remains synchronized and intelligent.

### 4.1. Discovery Process (On Startup / Periodic)

To adapt to changes in the target system, the Connector can run a discovery process.

1.  **Schema Fetch:** On startup or at a regular interval, it connects to the target system's API to fetch its OpenAPI schema, custom field definitions, or other metadata.
2.  **Internal Model Generation:** It parses this data to generate and cache internal models (e.g., `writable_schema.json`, `custom_field_map.json`). These models power the transformation engine.

### 4.2. Pre-Flight Transformation Engine (On Data Modification)

When the Connector receives a create or update request, the payload should pass through this engine before any API call is made.

1.  **Semantic Rule Application:** Apply hard-coded business logic for complex data structures.
2.  **Custom Field Relocation:** Automatically move any custom fields into the correct nested dictionary required by the API.
3.  **Data Type Coercion:** Enforce correct data types (e.g., string to integer) based on the discovered schema.
4.  **Name-to-ID Resolution:** The most critical step. Resolve human-readable names (e.g., `cluster_name: "US-EAST-PROD"`) into the numeric IDs required by the API.
5.  **Final Execution:** Execute the final, syntactically perfect API call to the target system.

---

## 5. gRPC API Definition (`[acronym]_service.proto`)

The service contract is defined using Protocol Buffers. The API should be generic and object-oriented.

[proto]
syntax = "proto3";

package connector.v1;

// The main service for interacting with [Target System Name].
service [ConnectorName]Service {
  // Generic method to search for objects.
  rpc GetObjects(GetObjectsRequest) returns (GetObjectsResponse);

  // Generic method to create a single object.
  rpc CreateObject(CreateObjectRequest) returns (CreateObjectResponse);
  
  // Add other generic methods like UpdateObject, DeleteObject, etc.
}

message GetObjectsRequest {
  string object_type = 1; // e.g., "dcim/sites", "virtualization/virtual-machines"
  string filters_json = 2; // Key-value filters as a JSON string
}

message GetObjectsResponse {
  string objects_json = 1; // A list of matching objects as a JSON string
  string error_message = 2;
}

message CreateObjectRequest {
  string object_type = 1;
  string object_data_json = 2; // The logical object payload as a JSON string
}

message CreateObjectResponse {
  string created_object_json = 1;
  string error_message = 2;
}
[/proto]

---

## 6. Configuration (`config/settings.yaml`)

* **`grpc_server`**: Host and port settings.
* **`target_system`**:
    * `api_url`: The base URL for the [Target System Name] API.
    * `api_token`: The API token for authentication (loaded securely from env vars).
* **`discovery`**: (Optional) Configuration for the discovery process, like the refresh interval.
* **`logging`**: Configuration for application and transaction logs.

---

## 7. Dependencies

**Python Libraries:**
* `grpcio`, `grpcio-tools`
* `pyyaml`
* `[e.g., pynetbox, requests]`: The specific Python client library for the target system.

---

## 8. Logging & Health Checks

The connector must adhere to all SERA Ecosystem Standards:

* **Logging:** Implements the standard `app.log`, `app_err.log`, and `transaction.log` pattern.
* **Health Checks:** Exposes a standard `grpc.health.v1.Health/Check` gRPC endpoint that returns a `SERVING` status. This is critical for Kubernetes liveness and readiness probes.

---

## 9. Development & Deployment Checklist

- [ ] **Define API:** Create the `.proto` file with the required services and messages.
- [ ] **Generate Stubs:** Compile the `.proto` file to generate the Python gRPC stubs.
- [ ] **Implement Server:** Write the main Python server logic that implements the gRPC service methods.
- [ ] **Implement Transformation Engine:** Implement the core logic for discovery and payload transformation.
- [ ] **Implement Standards:** Add standardized configuration loading, logging, and the health check endpoint.
- [ ] **Containerize:** Create a `Dockerfile` to package the service.
- [ ] **Test:** Write unit/integration tests and test the running container by making calls with a gRPC client.
- [ ] **Register with SERA:**
    - [ ] Place the compiled Python stubs into a unique subdirectory in the shared `/assets/protos/` directory.
    - [ ] Create the `*.yaml` tool definition file for the Sera Orchestrator.