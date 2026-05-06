# NetBox Assistant (NBA) - Design Document

**Version:** 1.5.0
**Date:** 2026-05-06
**Author(s):** Martin Kecka, Gemini AI
**Category:** Assistant (A)
**Acronym:** NBA

---

## 1. Introduction & Purpose

The **NetBox Assistant (NBA)** is a specialized AI-driven module within the SERA ecosystem, designed to act as the primary "domain expert" for all NetBox automation tasks. It functions both as a standalone application with a React-based UI for direct interaction and as a gRPC-enabled tool for the Sera Orchestrator.

Its core purpose is to translate high-level, natural language user requests (e.g., "Provision a new VM for the web team") into a complete, logically sound, multi-step execution plan. It then orchestrates the execution of this plan by making a series of calls to its deterministic counterpart, the **NetBox Connector (NBC)**. This architecture allows the NBA to handle the complex reasoning, planning, and error correction inherent in automation, while delegating the syntactically perfect, low-level API interactions to the NBC.

---

## 2. Architecture & Integration

* **Service Type:** (A) Assistant Module with a user-facing React UI.
* **Called by:**
    * **Sera Orchestrator:** The primary consumer in a production environment. The Orchestrator delegates complex NetBox-related goals to the NBA.
    * **Human User:** Via the React front-end application served by the FastAPI backend for testing, demonstration, and direct interaction.
* **Interacts with:**
    * **LLM Service (AI Factory):** The NBA is heavily reliant on the `gemma-4-31B-it` model running on vLLM for its reasoning capabilities across multiple stages of its pipeline.
    * **NetBox Connector (NBC):** The NBA's exclusive tool for interacting with NetBox. All plan execution steps are translated into specific, simple gRPC calls to the NBC.
    * **Internal Schema Files:** Reads the set of schema files (`object_map.json`, `writable_schema.json`, etc.) located in its `/discovery` directory. These files, which are kept up-to-date by the NBCClient, provide critical context to the LLM during plan generation.
* **Authentication:** Utilizes Keycloak for fetching OAuth2 tokens via Client Credentials grant to authenticate against the vLLM AI Factory endpoints.
* **Communication Protocol (UI to Backend):** Uses standard REST endpoints for intent and plan generation, but utilizes **Server-Sent Events (SSE)** for execution and dry-runs to stream real-time progress of long-running, multi-step operations to the frontend.

---

## 3. Core Functionality & Process Flow ⚙️

The NBA operates on a sophisticated, multi-stage pipeline that transforms a user's query into an executed action in NetBox. This entire process is orchestrated by the Execution Engine via the FastAPI router.

### 3.1. Stage 1: Intent Recognition & Recipe Selection

* **Intent Recognition:** The user's query is sent to the LLM using the `get_intent_recognition_prompt`. The LLM's task is to determine the user's primary goal (e.g., `provision_full_vm`) and extract all relevant information into a structured JSON `entities` object.
* **Recipe Selection:** Based on the recognized intent, the Planner deterministically selects a "recipe" from the internal `RECIPE_BOOK`. A recipe is a high-level, human-readable sequence of steps that guides the next stage (e.g., "Find the cluster," "Create the VM," "Create interfaces").

### 3.2. Stage 2: Execution Plan Generation

* **Prompt Assembly:** The NBA assembles a detailed prompt for the LLM using `get_plan_generation_prompt`. This prompt includes the original query, the structured entities, the selected recipe, and crucial context from the local schema files (e.g., a list of valid `object_type` values).
* **Plan Creation:** The LLM processes this rich context and generates a complete, multi-step JSON `execution_plan`. Each step in the plan contains a `tool` to use (e.g., `GetObjects`, `CreateObject`), the `params` for that tool, and a human-readable `description`. The plan uses placeholders like `{{steps[0].result[0].id}}` to chain dependencies between steps.

### 3.3. Stage 3: Dry Run & User Confirmation

* **Write Operation Check:** Before execution, the plan is inspected by the engine.
* **Read-Only Plan:** If the plan contains only read operations (`GetObjects`), it is executed immediately. Agentic loop plans are also routed here for read-only workflows.
* **Write Plan:** If the plan involves creating, updating, or deleting objects, the NBA triggers a safety protocol:
    1.  It performs a **dry run** which resolves all dependencies and simulates the write payloads without committing them. The progress of this simulation is streamed to the UI via SSE.
    2.  These simulated payloads are sent to the LLM with `get_dry_run_synthesis_prompt` to generate a clear, human-readable summary of the intended changes.
    3.  This summary is presented to the user in the UI for **explicit confirmation** before any action is taken.

### 3.4. Stage 4: Plan Execution with AI-Powered Correction 🤖

Upon approval (or immediately for read-only plans), the execution engine iterates through each step of the plan. Because this stage is unpredictable in duration, it operates as a generator, yielding its status to the client over an SSE stream (e.g., yielding "executing", then "healing", etc.).

* **Parameter Resolution:** It resolves any placeholders in the step's `params` using the results from previous steps.
* **gRPC Call:** It sends the resolved payload to the NetBox Connector (NBC) via a gRPC call.
* **AI Failure Analysis (Self-Correction):** If a gRPC call returns an error, the NBA does not immediately fail. Instead, it activates its self-correction mechanism:
    1.  The NBC returns a highly descriptive error, explicitly listing valid core fields and custom fields derived from the `writable_schema.json` (e.g., "Invalid field 'host'... Valid core fields are: ['cluster', 'device', 'name'...]").
    2.  The full plan, the failed step, and this schema-aware error message are sent to the LLM using the `get_failure_analysis_prompt`.
    3.  The engine yields a "healing" state to the frontend stream so the user is aware of the AI intervention.
    4.  The LLM analyzes the context and can either:
        * **Propose a Correction:** Return a corrected `params` object for the failed step (e.g., mapping 'host' to the correct 'device' field). The NBA will then automatically retry the step with the corrected parameters.
        * **Identify Unrecoverable Error:** If the error is fundamental (e.g., a unique constraint violation), the LLM returns an `unrecoverable` status with a reason, and the NBA halts execution gracefully.

### 3.5. Stage 5: Final Answer Synthesis

If the entire plan executes successfully, the final data from the last step is sent to the LLM with the `get_synthesis_prompt`. The LLM generates a concise, natural-language summary of the result, which is presented to the user as the final answer.

---

## 4. User Interface (React)

The NBA includes a rich, interactive user interface built with React and Material UI for direct interaction and testing.

* **Input:** A text area allows the user to enter their natural language request. A primary submit button initiates the pipeline.
* **Real-Time Process Visualization:** As the pipeline runs, the UI provides real-time status updates for each stage. During execution and dry-runs, a dynamic progress component decodes the SSE stream to display exact state tracking (e.g., "Execution Step 2 [UpdateObject] (Retry 2)...").
* **Transparency & Accordions:** The UI includes expandable sections that allow the user to inspect the raw JSON output and operations at key stages:
    * **Intent Analysis:** Shows the recognized intent and extracted entities.
    * **Execution Plan:** Displays the full multi-step plan before it's executed, including simulated write payloads.
    * **AI Self-Healing Applied:** A dedicated warning-colored accordion prominently displays any errors encountered and the exact JSON payload corrections the AI injected to fix them mid-flight.
    * **Live Logs:** Each step of the execution is shown in its own expander, displaying the gRPC request payload and the resulting data or error.
* **Confirmation Prompt:** For any plan that will modify data in NetBox, the UI presents the AI-generated summary and requires the user to click a "Confirm & Execute" button, acting as a critical safety gate.

---

## 5. Configuration (config/settings.yaml)

The NBA is configured via its `config/settings.yaml` file, which includes:

* `nbc_service_config`: Host and port for the NetBox Connector (NBC) gRPC service it calls.
* `llm_interface`: The base URL, API endpoint, model name (`google/gemma-4-31B-it`), and context limits for the AI Factory's vLLM service.
* `auth`: Keycloak configuration (Auth URL, Client ID, Client Secret) for securing calls to the LLM backend.
* `logging`: Configuration for application and error log files.
* `transaction_logging`: Enables/disables and configures the per-run transaction logs.
* `paths`: Defines the shared assets path for gRPC stubs.
* `user_context`: Default user information for local testing.

---

## 6. Dependencies

* **External Services:**
    * A running NetBox Connector (NBC) instance.
    * A running vLLM endpoint serving `gemma-4-31B-it` in the AI Factory.
    * Keycloak server for OAuth2 client credentials authentication.
* **Python Libraries:** `fastapi`, `uvicorn`, `grpcio`, `pyyaml`, `requests`, `tenacity`.
* **Frontend:** `react`, `@mui/material`, `vite`.

---

## 7. Logging & Auditing

The NBA implements a robust logging strategy for observability and auditing.

* **Application Logging:** Standard service lifecycle events are logged to `nba_app.log` and `nba_err.log` as configured in `utils/logging_setup.py`.
* **Transaction Logging:** This is the core auditing feature. The `PipelineTransactionLogger` creates a unique, timestamped JSON file for every single pipeline run. This log file captures the entire lifecycle of a request, including the initial user query, the AI-generated plan, the resolved payload for each executed step, the result or error from the NBC, the specific self-healing interventions applied, and the final synthesized answer. This provides an exhaustive, step-by-step audit trail for every action the assistant takes.

---

## 8. Future Enhancements

* **Full gRPC Service Implementation:** Formalize the gRPC service interface for the NBA itself, allowing it to be fully integrated as a tool in the SERA Orchestrator according to the `tools.d/*.yaml` contract.
* **Stateful, Long-Running Tasks:** Enhance the architecture to manage state across multiple interactions, allowing for clarification questions to the user mid-execution (e.g., "I found two sites named 'US-WEST', which one should I use?").
* **Dynamic Recipe Generation:** Explore using RAG to dynamically generate recipes from internal process documentation, rather than relying on the static, hard-coded `RECIPE_BOOK`.
* **Visual Plan Representation:** Enhance the React UI to display the execution plan as a visual graph or flowchart, making complex dependencies easier to understand.
* **Self-Improving Failure Analysis with a Correction Cache:** Create an adaptive feedback loop to reduce errors from the plan-generation LLM over time. 
    * **Implement a Correction Cache (`fix_schema.json`):** While the NetBox Connector currently provides excellent, schema-aware error messages that allow the LLM to successfully heal failures on the fly, this still requires an extra API call. Whenever the Failure Analysis engine successfully corrects a payload (e.g., mapping `host` to `device`), it will record this successful transformation as a rule in a new persistent `fix_schema.json` cache.
    * **Establish a Feedback Loop:** During the initial Execution Plan Generation stage, the contents of `fix_schema.json` will be injected into the LLM prompt. This provides the planning LLM with a set of high-priority, learned instructions based on previously corrected errors, significantly increasing the probability that the initial plan is correct on the first attempt and reducing the need for reactive corrections.
* **Intelligent Post-Processing with a Data Analysis Engine:** Address the challenge of handling large datasets returned by `GetObjects` calls, which can be inefficient or too large for an LLM's context window. This involves creating a new data analysis step in the pipeline.
    * **Introduce a New `AnalyzeData` Tool:** Create a new internal tool powered by the `Pandas` library. This tool will be designed to take a large list of JSON objects (the output of a `GetObjects` call) and load it into a `DataFrame` for high-performance, in-memory processing.
    * **Enhance the Planner:** The Intent Recognition stage will be trained to identify queries that require post-filtering or analysis (e.g., "list all VMs where custom field X is empty"). For such queries, the Execution Plan Generation stage will create a two-step plan: a `GetObjects` step to fetch the broad dataset, followed by an `AnalyzeData` step.
    * **AI-Generated Analysis Steps:** The planning LLM will generate structured parameters for the `AnalyzeData` tool, instructing it on how to filter, sort, or aggregate the data. For example, for the query "list all VMs in cluster X without a 'cs_group' custom field", the LLM would generate an `AnalyzeData` step with parameters like `{"input": "{{steps[0].result}}", "filter": "custom_fields.cs_group == null"}`.
    * **Efficient Synthesis:** The final, pre-processed `DataFrame` (which may be a much smaller list of VMs or even just a single count) is then passed to the Final Answer Synthesis stage. This ensures the LLM receives only the most relevant information, making its summary faster, more accurate, and more reliable.
* **Human-in-the-Loop Guided Correction 🧑‍🏫:** Transform the failure analysis process into a powerful, interactive teaching mechanism that complements the automated correction cache.
    * **Introduce an Intervention Step:** When an execution step fails, the React UI will pause the SSE stream and present the user with the failed step, the schema-aware error message from the NBC, and the AI's proposed correction from the Self-Healing engine.
    * **Empower the User:** The user will have the option to Accept the AI's fix or Provide a Manual Correction by editing the payload themselves directly in the UI.
    * **Create a Teaching Loop:** If the user provides a manual fix, that human-validated correction is not only used to retry the step but is also automatically saved as a new rule in the `fix_schema.json` Correction Cache. This creates a direct feedback channel where human expertise is used to explicitly train the assistant, ensuring it learns the correct solution and improving the quality of future, fully autonomous plan generations.