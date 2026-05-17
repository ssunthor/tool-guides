# ServiceNow Connector (SNC) - Design & Architecture Document

**Version:** 0.1.2
**Date:** 2026-05-17
**Category:** (C) Connector
**Acronym:** SNC

---

## 1. Introduction & Purpose

The **ServiceNow Connector (SNC)** is a deterministic gateway module designed to abstract the complexities, slowness, and inconsistencies of the native ServiceNow REST API. It exposes a clean, strongly-typed **gRPC interface** and a comprehensive **CLI** for downstream AI systems, administrators, and UI components (like the SAM dashboard).

Instead of forcing downstream systems to understand ServiceNow's underlying table structures (e.g., `sc_req_item` vs `sc_task`), `sys_id` resolution, or variable dictionaries, SNC handles all translations, ID lookups, and fast-caching locally.

---

## 2. Core Architecture

The module is built on a standard Python gRPC architecture:

* **`core/snow_client.py`:** The heart of the connector. Handles OAuth2 Client Credentials authentication, session pooling, error handling, and the direct `requests` translation to ServiceNow's Table and Service Catalog APIs.
* **`grpc_api/services.py`:** The gRPC server implementation mapping incoming RPC requests (`snc_service.proto`) to the underlying `SnowClient` methods.
* **`cli/snc_cli.py`:** A robust Click-based CLI that consumes the local gRPC server, allowing administrators and developers to test integrations, pull statistics, and trace tickets directly from the terminal.
* **`utils/rpc_transaction_logger.py`:** Captures every single inbound gRPC request and outbound response, dumping them as JSON for auditability and debug tracing.

---

## 3. Key Subsystems & Features

### 3.1. Standard Ticket Management (`ticket` & `group` commands)
* **Unified Abstraction:** Pulls records from standard tables (`incident`, `sc_task`, `problem`) and translates complex reference fields (like `assigned_to` or `assignment_group`) from raw `sys_id`s into human-readable display values.
* **Public vs Private Notes:** Downstream systems can seamlessly pass a boolean flag to write to internal `work_notes` (visible only to IT) or customer-visible `comments` (which trigger email notifications to the caller).
* **Queue Statistics:** Can dynamically pull ticket states, ages, and assignee distribution for specific engineering groups.
* **Deep Tracing:** Can resolve hierarchical ticket trees (e.g., finding all child `RITM`s and `SCTASK`s associated with a parent `REQ`).

### 3.2. Explicit ITIL Domain Lifecycles (`incident` & `request` commands) [NEW in v0.1.2]
To simplify integrations for upstream applications and AI orchestration layers, SNC features strongly-typed domain handlers that bypass the generic request pipelines:
* **Incident Lifecycle Management:** Offers clean wrappers for explicit generation, field reading, operational update tracking, and strict resolution compliance tracking on the raw `incident` table.
* **Service Catalog Domain Handlers:** Provides isolated interfaces to submit, view, update, and close item-specific catalog fulfillment structures, mapping inputs directly to tracked task strings (`SCTASK`/`RITM`).

### 3.3. Service Catalog Registry (`req` commands)
ServiceNow's Service Catalog API is notoriously slow and heavily nested. SNC solves this via a local caching registry.
* **Fast-Sync:** SNC downloads a complete copy of all active catalog items and stores them in `static/catalog_registry.json`.
* **Instant Search:** Downstream systems query the local JSON file instead of ServiceNow, allowing for instant semantic searches across thousands of ITIL request items.
* **Variable Extraction:** Automatically recursively parses ServiceNow's containerized catalog variables to return a flattened dictionary of inputs required to submit a specific request. It specifically queries the `sc_item_option_mtom` table to fetch variables tied to an ordered ticket.
* **Dynamic Endpoint Routing:** When an order is placed, SNC checks the registry. If it is a Record Producer (like "Create Incident"), it automatically routes the payload to the `/submit_producer` endpoint. If it is a standard item, it routes to `/order_now`.
* **Auto-Tracing:** When a standard catalog item is ordered, SNC waits 2 seconds for SNOW's business rules to fire, traces the generated hierarchy, and returns the exact actionable `RITM` and `SCTASK` IDs directly in the order response.

### 3.4. Smart Identity Resolution
Downstream users interact with SNC using email addresses or usernames. SNC automatically queries the `sys_user` and `sys_user_group` tables to resolve these human-readable identifiers into the exact 32-character `sys_id` hashes required by ServiceNow payloads. 

To prevent API rate-limiting and latency, SNC implements a **Persistent Identity Cache** (`static/identity_cache.json`). 
* If a downstream app sends "martin.kecka@gtt.net", SNC resolves the `sys_id` via the API, caches the mapping to disk, and uses the `sys_id` in the final payload.
* When ServiceNow returns a raw `sys_id` (such as inside a Catalog Variable), SNC intercepts it, checks the cache, and attempts to return the human-readable display name back to the downstream application.

---

## 4. Static Overrides & Guardrails

To prevent hardcoding business logic into the Python codebase, SNC relies on JSON-based static mappings:

### 4.1. `state_mappings.json`
ServiceNow represents ticket states as integers (e.g., `1`, `2`, `3`), but the meaning of these integers changes depending on the table. `state_mappings.json` acts as a translation layer, allowing users to send `"state": "work in progress"` while SNC translates it to `2` for `sc_task` or whatever the equivalent is for an `incident`.

### 4.2. `catalog_overrides.json`
Certain Service Catalog items possess complex UI Policies or Client Scripts that the REST API bypasses or misinterprets. `catalog_overrides.json` allows developers to define mandatory variables locally or force user-resolution on specific keys (e.g., mapping a raw email to a `sys_id` for a `requested_for` or `assignment_group` variable) to ensure catalog orders succeed via the API without triggering 500 Internal Server Errors.

### 4.3. `resolution_mappings.json`
When resolving an Incident, ServiceNow requires a specific internal `close_code`. However, the internal database values rarely match the human-readable labels presented in the UI. This mapping file acts as a translation layer, allowing downstream users to send standard strings (e.g., `"permanent fix"`) which SNC translates into the correct database code (e.g., `"Resolved - Permanent Fix Applied"`) to prevent the system from auto-defaulting to "Resolved by caller".

---

## 5. Deployment & Usage

### 5.1. Starting the Daemon
The gRPC server runs in the background to serve traffic.
[bash]
python snow_connector.py start-grpc-server
[/bash]

### 5.2. CLI Examples
The CLI acts as the primary administrative interface and mirrors the exact capabilities of the gRPC stubs.

**Sync the catalog to the local cache**
[bash]
python snow_connector.py req sync
[/bash]

**Trace a ticket's entire hierarchy**
[bash]
python snow_connector.py ticket trace REQ0012345
[/bash]

**View real-time queue stats for a team**
[bash]
python snow_connector.py queue stats --group "Systems Engineering"
[/bash]

**Ordering a Record Producer (e.g., Creating an Incident)**
Record producers create flat records directly. Thanks to `catalog_overrides.json`, you can pass raw emails and group names.
[bash]
python3 snow_connector.py req order "Create Incident" \
  --vars '{
    "caller_id": "martin.kecka@gtt.net",
    "urgency": "3",
    "assignment_group": "Systems Engineering",
    "short_description": "API Integration Test",
    "description": "Testing Record Producer creation via the SNC gateway."
  }'
[/bash]

**Ordering a Standard Catalog Item (Hierarchical)**
Standard items generate a `REQ`. The SNC gateway will automatically trace the request and return the child tasks.
[bash]
python3 snow_connector.py req order "Generic System Engineering Request" \
  --vars '{
    "requested_for": "martin.kecka@gtt.net",
    "gtt_business_justification": "Testing the automated provisioning flow.",
    "priority": "3",
    "description": "Please provision the requested test environment."
  }'
[/bash]

**Adding Customer-Visible Comments vs Internal Notes**
Use the `--public` flag to route the note to the `comments` field instead of internal `work_notes`.
[bash]
# Internal Note (Engineers only)
python3 snow_connector.py ticket add-note INC0147804 "Investigating the API test incident internally."

# Public Comment (Customer gets an email)
python3 snow_connector.py ticket add-note INC0147804 "We are working on your issue now." --public
[/bash]

**Resolving a Ticket**
Resolving an Incident requires a state change, a resolution code, and close notes.
[bash]
python3 snow_connector.py ticket update INC0147804 \
  --state "resolved" \
  --resolution-code "permanent fix" \
  --notes "API Integration Test successfully completed and verified."
[/bash]

### 5.3. Explicit Lifecycle Domain Commands

**Creating a New Incident Directly**
[bash]
python3 snow_connector.py incident create \
  --caller "martin.kecka@gtt.net" \
  --short-desc "Network Switch Disconnect" \
  --desc "Core switch in Rack B is showing intermittent link failure." \
  --category "Systems Engineering" \
  --subcategory "Other" \
  --urgency "2" \
  --group "Systems Engineering"
[/bash]

**Querying an Incident with Raw gRPC Inspection Enabled**
[bash]
python3 snow_connector.py incident get INC0147742 --grpc
[/bash]

**Resolving an Incident with Typed Arguments**
[bash]
python3 snow_connector.py incident resolve INC0147742 \
  --code "permanent fix" \
  --notes "Replaced damaged fiber patch cable on Port 12."
[/bash]

**Submitting a Service Request**
[bash]
python3 snow_connector.py request create "Generic System Engineering Request" \
  --vars '{"requested_for": "martin.kecka@gtt.net", "priority": "3", "description": "Provisioning dev stack."}'
[/bash]

**Closing a Specific Request Task (SCTASK or RITM)**
[bash]
python3 snow_connector.py request resolve SCTASK0044457 --notes "Environment built and verified."
[/bash]