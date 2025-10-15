# [Assistant Name] ([Acronym]) - Design Document

**Version:** 1.0
**Date:** [Date]
**Author(s):** [Author Name]
**Category:** Assistant (A)
**Acronym:** [Acronym]

---

## 1. Introduction & Purpose

The **[Assistant Name] ([Acronym])** is a specialized AI-driven module within the SERA ecosystem, designed to act as the "domain expert" for **[Target Domain, e.g., NetBox, CloudStack]** automation tasks.

Its core purpose is to translate high-level, natural language user requests into a complete, logically sound, multi-step execution plan. It then orchestrates the execution of this plan by making a series of calls to one or more deterministic **(C) Connector** modules. This architecture allows the Assistant to handle complex reasoning, planning, and error correction, while delegating the simple, low-level API interactions to the Connectors.

---

## 2. Architecture & Integration

* **Service Type:** (A) Assistant Module, callable as a tool by the Sera Orchestrator and optionally including a standalone UI for direct interaction.
* **Called by:**
    * **Sera Orchestrator:** The primary consumer, which delegates complex goals to this assistant.
    * **Human User:** Can be launched with a Streamlit UI for testing, demonstration, and direct interaction.
* **Interacts with:**
    * **LLM Service (AI Factory):** Relies on a powerful reasoning model (e.g., `gemma3:27b`) for its core pipeline stages.
    * **[Connector Name] ([Acronym]):** Uses one or more Connector modules as its exclusive tools for interacting with target systems.
    * **Internal Knowledge/Schema Files:** May use local files (e.g., schemas, process "recipes") to provide critical context to the LLM during plan generation.

---

## 3. Core Functionality & Process Flow ⚙️

The Assistant operates on a multi-stage pipeline that transforms a user's ambiguous query into a precise, executed action. This establishes a robust and repeatable pattern for AI-driven automation.

### 3.1. Stage 1: Intent Recognition & Recipe Selection

* **Intent Recognition:** The user's query is sent to the LLM to determine the primary goal (e.g., `provision_vm`) and extract all relevant information into a structured JSON `entities` object.
* **Recipe Selection:** Based on the recognized intent, the Assistant deterministically selects a "recipe" from an internal `RECIPE_BOOK`. A recipe is a high-level, human-readable sequence of steps that guides the next stage.

### 3.2. Stage 2: Execution Plan Generation

* **Prompt Assembly:** The Assistant assembles a detailed prompt for the LLM. This prompt includes the original query, the structured entities, the selected recipe, and any crucial context from local knowledge files.
* **Plan Creation:** The LLM processes this rich context and generates a complete, multi-step JSON `execution_plan`. Each step in the plan must contain a `tool` to use (a specific Connector function), the `params` for that tool, and a `description`. The plan should use placeholders (e.g., `{{steps[0].result.id}}`) to chain dependencies between steps.

### 3.3. Stage 3: Dry Run & User Confirmation (for Write Operations)

* **Write Operation Check:** Before execution, the plan is inspected for any operations that would modify data.
* **Read-Only Plan:** If the plan is read-only, it is executed immediately.
* **Write Plan:** If the plan involves creating, updating, or deleting data, the Assistant must trigger a safety protocol:
    1.  It performs a **dry run**, resolving all dependencies and simulating the final payloads without committing them.
    2.  The simulated payloads are sent to the LLM to generate a clear, human-readable summary of the intended changes.
    3.  This summary is presented to the user for **explicit confirmation** before any action is taken.

### 3.4. Stage 4: Plan Execution with AI-Powered Correction 🤖

Upon user approval, the Assistant iterates through each step of the plan.

* **Parameter Resolution:** It resolves any placeholders in the step's `params` using results from previous steps.
* **gRPC Call:** It sends the resolved payload to the appropriate Connector via a gRPC call.
* **AI Failure Analysis (Self-Correction):** If a Connector returns an error, the Assistant must activate a self-correction mechanism:
    1.  The full plan, the failed step, and the exact error message are sent to the LLM for analysis.
    2.  The LLM can either **propose a correction** (which the Assistant automatically retries) or identify the error as **unrecoverable**, halting the process gracefully and informing the user.

### 3.5. Stage 5: Final Answer Synthesis

If the plan executes successfully, the final data is sent to the LLM, which generates a concise, natural-language summary of the result for the user.

---

## 4. gRPC API Definition (`[acronym]_service.proto`)

While primarily called by the Orchestrator via the `tools.d` contract, the Assistant should expose a formal gRPC service to allow for other forms of programmatic interaction.

[proto]
syntax = "proto3";

package assistant.v1;

// The main service for the [Assistant Name].
service [AssistantName]Service {
  // Processes a high-level natural language request.
  // The response is streamed to provide real-time updates on the pipeline's progress.
  rpc ProcessNaturalLanguageRequest(NLRequest) returns (stream NLResponse);
}

message NLRequest {
  string user_query = 1;
  // Optional metadata like user ID, session ID, etc.
}

message NLResponse {
  enum Stage {
    // Enum values for each stage of the core process flow
  }
  Stage current_stage = 1;
  string message = 2; // Human-readable status update
  string payload_json = 3; // JSON data for plan, result, etc.
}
[/proto]

---

## 5. User Interface (Optional but Recommended)

For development, testing, and standalone use, the Assistant should include an interactive UI built with Streamlit. The UI should provide:

* **Input:** A text area for the natural language request.
* **Process Visualization:** Real-time status updates as the pipeline runs.
* **Transparency:** Expandable sections to inspect the raw JSON at each stage (intent, plan, execution steps).
* **Confirmation Prompt:** A clear dialog requiring a user to click "Confirm & Execute" for any write operations.

---

## 6. Configuration (`config/settings.yaml`)

* **`grpc_server`**: Host and port for the Assistant's own gRPC service.
* **`[connector_acronym]_service_config`**: Host and port for the Connector(s) it depends on.
* **`llm_interface`**: Endpoint, model name, and timeout for the reasoning model.
* **`logging` & `transaction_logging`**: Standard SERA logging configuration.

---

## 7. Logging & Auditing

The Assistant must implement a robust logging and auditing strategy.

* **Application Logging:** Standard service events are logged to `[acronym].log` and `[acronym]_err.log`.
* **Transaction Logging:** A `PipelineTransactionLogger` must be used. For every single run, it must create a unique, timestamped JSON log file capturing the entire request lifecycle: query, intent, plan, each execution step's payload and result/error, and the final synthesized answer. This provides an exhaustive audit trail.

---

## 8. Dependencies

* **External Services:**
    * A running instance of the required **(C) Connector(s)**.
    * A running LLM endpoint in the AI Factory.
* **Python Libraries:** `streamlit`, `grpcio`, `pyyaml`, etc.

---

## 9. Development & Deployment Checklist

- [ ] **Define API:** Create the `.proto` file for the Assistant's gRPC service.
- [ ] **Generate Stubs:** Compile the `.proto` file.
- [ ] **Implement Core Pipeline:** Write the Python code for the five-stage process flow.
- [ ] **Implement gRPC Server:** Implement the gRPC service logic.
- [ ] **Build UI:** Create the standalone Streamlit interface for testing.
- [ ] **Implement Standards:** Add standardized configuration, logging (including transaction logging), and health checks.
- [ ] **Containerize:** Create a `Dockerfile`.
- [ ] **Test:** Write unit/integration tests and perform end-to-end tests using the UI.
- [ ] **Register with SERA:**
    - [ ] Place compiled gRPC stubs into a unique subdirectory in `/assets/protos/`.
    - [ ] Create the `*.yaml` tool definition file for the Sera Orchestrator.