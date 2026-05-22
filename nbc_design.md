# NetBox Connector (NBC) - Design Document

**Version:** 1.1
**Date:** 2025-08-21
**Author(s):** Martin Kecka, Gemini AI
**Category:** Connector (C)
**Acronym:** NBC

---

## 1. Introduction & Purpose

The **NetBox Connector (NBC)** is a specialized gRPC service module within the SERA AI ecosystem. It serves as the single, authoritative gateway for all interactions with the NetBox API. Its primary purpose is to act as a deterministic expert tool for higher-level modules, particularly the NetBox Assistant (NBA).

The NBC solves a critical architectural challenge: the fragility of AI-generated API payloads. Instead of requiring the NBA to be an expert in NetBox's specific, often complex syntax, the NBC accepts simplified, logical requests. It then uses a schema-driven, multi-stage **Pre-Flight Transformation Engine** to convert these logical requests into syntactically perfect, ID-based REST API calls that NetBox is guaranteed to understand. This separation of concerns creates a more resilient, scalable, and maintainable automation suite.

---

## 2. Architecture & Integration

**Service Type:** External gRPC Service.

**Called by:** Primarily the **NetBox Assistant (NBA)**, which formulates logical plans. The call is facilitated by the Sera Orchestrator's `ToolExecutor`.

**Interacts with:**
- **NetBox REST API:** The target system of truth for all CRUD operations.
- **Internal Schema Files:** Reads a set of self-generated JSON files (`writable_schema.json`, `filterable_schema.json`, `custom_field_map.json`) located in its `/discovery` directory. These files are the "brain" that powers its transformation engine.

---

## 3. Core Functionality & Process Flow

The NBC's functionality is divided into two key processes: the initial **Discovery Process** that builds its intelligence, and the runtime **Pre-Flight Transformation Engine** that uses that intelligence.

### 3.1. The Discovery Process (Startup & Background)

This process runs automatically to ensure the NBC is always synchronized with the target NetBox instance's capabilities.

1.  **Version Check:** On startup, the service calls the NetBox `/api/status/` endpoint to get the current `netbox-version`.
2.  **Conditional Full Discovery:** It compares this version to a cached version in `discovery/netbox_version.cache`. If the version has changed (or the cache doesn't exist), it triggers a full discovery:
    -   Fetches the entire OpenAPI schema from NetBox.
    -   Parses the schema to generate `writable_schema.json` (defining create/update payloads) and `filterable_schema.json` (defining valid query parameters).
    -   Fetches all custom fields and their choice sets to generate `custom_field_map.json`.
    -   Saves the new NetBox version to the cache file.
3.  **Periodic Custom Field Refresh:** A background thread runs at a configurable interval (e.g., every hour). This thread re-runs only the custom field discovery process, ensuring the NBC adapts to any changes in custom fields without requiring a service restart.

### 3.2. Pre-Flight Transformation Engine (On CreateObject/UpdateObject)

When the NBC receives a `CreateObject` or `UpdateObject` request, the `object_data` payload passes through this engine before any API call is made.

1.  **Semantic Rule Application:** The engine applies hard-coded logic for complex object structures. For example, if it receives `{"mode": "access", "vlan": "CENTRAL-MGMT", "virtual_machine": ...}`, its "interface_mode_logic" rule triggers. It uses the `virtual_machine` ID to find the VM's site, then finds the unique VLAN named "CENTRAL-MGMT" within that site, and finally restructures the payload to use the correct `untagged_vlan` key with the resolved VLAN ID.
2.  **Custom Field Relocation:** It cross-references all top-level keys in the payload against `custom_field_map.json`. Any key identified as a custom field is automatically moved into the nested `custom_fields: {}` dictionary.
3.  **Data Type Coercion & Choice Normalization:** Using `writable_schema.json`, it enforces correct data types (e.g., converting a string "2" to an integer 2 for `vcpus`) and normalizes choice field values (e.g., converting a display name "Active" to the required API value "active" for `status`).
4.  **Name-to-ID Resolution:** For all remaining relationship fields (e.g., `cluster`, `role`, `tenant`), it resolves the provided human-readable string names into their corresponding numeric IDs. This process includes context-aware disambiguation to resolve ambiguous names.
5.  **Final Execution:** Only after all transformations are complete does the NBC execute the final, syntactically perfect REST API call to NetBox.

---

## 4. gRPC API Definition 📜

The service exposes a clear, function-oriented API using gRPC. The definition below is an illustrative overview. For the complete and authoritative contract, please refer to the source `.proto` file.

**Source File:** `netbox_connector/grpc_api/protos/nbc_service.proto`
**Service:** `NetBoxConnectorService`

---

### Schema & Metadata Methods

- **GetSchemaVersions:** A lightweight health check that returns the current version of the NetBox instance and a hash of the custom field map. Used to check if schemas have changed.
- **GetLogicalSchemas:** A heavier call that returns the full JSON schemas (`writable_schema`, `filterable_schema`, etc.) that the NBC uses for its transformation logic.

---

### Data Management Methods

- **GetObjects:** A generic method to search for any object type.
    - **Input:** `object_type` (e.g., "dcim/sites"), `filters` (a JSON object of key-value pairs), and an optional list of `output_fields`.
    - **Output:** A list of matching objects.
- **CreateObject:** Creates a single new object of any type.
    - **Input:** `object_type` and the logical `object_data` payload.
    - **Output:** The full object as created in NetBox.
- **UpdateObject:** Modifies an existing object.
    - **Input:** `object_type`, the `object_id` to modify, and an `object_data` payload containing only the fields to be changed.
    - **Output:** The full, updated object.
- **DeleteObject:** Deletes an object by its ID.
    - **Input:** `object_type` and `object_id`.
    - **Output:** A success/failure confirmation.
- **GetNextAvailableIp:** A specialized method for IPAM.
    - **Input:** The numeric `prefix_id`.
    - **Output:** The next available IP address object within that prefix.

---

## 5. Configuration (`config/settings.yaml`)

The NBC is configured via its `config/settings.yaml` file, which includes:

-   `grpc_server`: Host and port for the service to listen on.
-   `target_system`: The URL and API token for the target NetBox instance.
-   `discovery`: The interval in seconds for the periodic custom field discovery.
-   `logging`: Configuration for application and error log files.
-   `transaction_logging`: Enables/disables and configures per-request transaction logs.
-   `paths`: Defines paths for shared assets and generated gRPC stubs.

---

## 6. Dependencies

-   **External Services:** A running NetBox instance with a valid API endpoint and token.
-   **Python Libraries:** `grpcio`, `requests`, `pyyaml`, `packaging`, etc. The service is self-contained and does not have dependencies on other SERA modules to perform its core function.

---

## 7. Logging & Auditing

The service implements the SERA standard for multi-file logging:

-   **Application Logging:** General service lifecycle events (startup, errors, warnings) are logged to `nbc.log` and `nbc_err.log` in the configured log directory. This is for monitoring the health of the service itself.
-   **Transaction Logging:** When enabled, every gRPC call is recorded in a separate, timestamped JSON file within the `transactions` subdirectory. Each file contains the full request payload, the final response from NetBox, and the success/error status, providing a complete audit trail for debugging and analysis.

---

## 8. Future Enhancements

-   **Advanced Semantic Rules:** Expand the Pre-Flight Transformation Engine with more complex, hard-coded business logic for other common NetBox object types (e.g., circuit terminations, power feeds).
-   **Webhook Support:** Develop an endpoint to listen for webhooks from NetBox. This would allow the NBC to proactively invalidate or update its internal caches (e.g., the ID cache) in real-time when an object is changed directly in NetBox, rather than waiting for a cache miss.
-   **Batch Operations:** Implement `BulkCreateObjects` and `BulkUpdateObjects` methods that can process a list of objects in a single gRPC call, allowing for more efficient bulk operations.