# /ai/assets/docs/tool_guides/raa_design.md
## Remote Automation Assistant (RAA) - Design Document

**Version:** 0.2
**Date:** 2025-08-12  
**Author(s):** Martin Kecka, Gemini AI
**Category:** Assistant (A)
**Acronym:** RAA (Remote Automation Assistant)

---

## 1. Introduction & Purpose

### 1.1. Overview

The **Remote Automation Assistant (RAA)** is a specialized (A) Assistant module within the SERA ecosystem. Its primary function is to serve as an intelligent planner and orchestrator for infrastructure automation tasks.

The RAA translates natural language requests into concrete, multi-step action plans. It operates in two distinct modes:

* **Dynamic Composition:** For novel or simple tasks, it directly calls and chains together various independent (C) **Connectors** (e.g., NetBox Connector, CloudStack Connector) to achieve a goal.
* **Delegated Execution:** For complex, sensitive, or high-stakes tasks, it triggers pre-defined, audited workflows on the (F) **Remote Automation Framework (RAF)**.

### 1.2. Key Objectives

* **Intelligent Planning:** Use LLMs to interpret natural language requests and formulate a sequence of steps required to complete the task.
* **Capability Discovery:** Dynamically discover the capabilities of all available tools, including the atomic functions of (C) Connectors and the high-level workflows of the (F) Framework.
* **Flexible Execution:** Seamlessly switch between calling individual connectors for granular control and delegating to the RAF for robust, pre-approved processes.
* **Context Management:** Gather, manage, and pass the necessary information between tool calls, ensuring each step in the plan has the data it needs.
* **AI-Assisted Generation:** Facilitate the AI-driven generation of simple report configurations or automation snippets that can be passed to the RAF for secure execution.
* **Result Interpretation:** Process the outputs from both Connectors and the RAF, consolidating them into a coherent result for the SERA Orchestrator.

### 1.3. Relationship with Other Modules

* **RAA as an Orchestrator:** RAA acts as a "sub-orchestrator" within SERA, focused solely on the domain of infrastructure automation. It receives a high-level goal and is responsible for the entire lifecycle of planning and executing the solution.
* **RAA as a Client:** RAA is a primary client to a wide range of modules:
    * Multiple (C) Connectors for performing atomic actions.
    * The (F) Remote Automation Framework (RAF) for executing complex, secure workflows.
* **Separation of Concerns:**
    * **SERA Orchestrator:** Manages the main user interaction and delegates automation goals to the RAA.
    * **RAA:** The "planner" and "thinker". It decides what to do and how to do it by creating an action plan.
    * **Connectors:** The "doers" for simple, atomic tasks (e.g., `get_ip`, `create_ticket`).
    * **RAF:** The "doer" for complex, pre-defined, and audited tasks (e.g., `provision_vm_workflow`).

---

## 2. RAA Architecture & Core Components

As a module within SERA, RAA's architecture consists of several key classes:

* **RAA Processor (Main Class):**
    * The central coordinating class. Receives high-level goals from the SERA Orchestrator.
    * Orchestrates the entire process: planning the task, discovering capabilities, executing the plan via the Tool Executor, and processing the final results.

* **Capability Manager:**
    * Responsible for discovering and caching the capabilities of all available tools that are tagged for RAA's use.
    * It queries (C) Connectors to understand their atomic functions (e.g., `netbox_connector.get_device`).
    * It queries the (F) Framework (RAF) to understand its available high-level workflows (e.g., `raf.list_workflows`).
    * This information is compiled and provided to the LLMTaskFormulator to create accurate plans.

* **LLM Task Formulator:**
    * The "brain" of the RAA. It utilizes an LLM to translate a natural language request into a structured, multi-step plan.
    * The plan consists of a precise sequence of gRPC calls to be made by the Tool Executor.
    * It determines whether to use Dynamic Composition (calling Connectors) or Delegated Execution (calling RAF) based on the user's request and the available tools.
    * For AI-Assisted Generation tasks (e.g., creating a new report script), it constructs the prompt to generate the necessary code/config and sends the result to the RAF for secure registration and execution.

* **Tool Executor:**
    * A generic gRPC client executor, similar to the one in the main SERA Orchestrator.
    * It dynamically loads the gRPC stubs and definitions of all tools available to the RAA (from the central tool registry).
    * It is responsible for making the actual gRPC calls to the various (C) Connectors and the (F) Framework as dictated by the plan from the LLMTaskFormulator.

* **RAA Config Loader:**
    * Loads the RAA's configuration from SERA's central config files.
    * Crucially, it determines which tools from the central registry the RAA is allowed to use, likely based on a specific scope tag (e.g., `scope: infrastructure`).

---

## 3. Process Flow (Example: User requests a new VM)

This example illustrates how the RAA combines both Dynamic Composition and Delegated Execution.

1.  **User Request (to SERA):** "Provision a new web server in the production zone using the standard template. Please create a change request for it first."
2.  **SERA Orchestrator:** Identifies this as an infrastructure automation task and delegates the entire goal to the RAA module.
3.  **RAA - Planning (LLMTaskFormulator):**
    * The RAA receives the request. The LLM analyzes the goal and the available tools (provided by the `CapabilityManager`).
    * It determines this is a multi-step process that involves gathering information before executing a complex, pre-defined task.
    * It formulates the following plan:
        * Call the ServiceNow Connector to create a new change request ticket. Store the resulting ticket number.
        * Call the NetBox Connector to find the next available IP address in the production VLAN. Store the IP address.
        * Call the RAF and trigger the `provision_vm` workflow, passing the stored ticket number and IP address as parameters.
4.  **RAA - Execution (Tool Executor):**
    * **Step 1 (Dynamic Composition):** The Tool Executor calls `servicenow_connector.create_change_request(summary="New Web Server")`. The connector returns `{"ticket": "CHG00123"}`.
    * **Step 2 (Dynamic Composition):** The Tool Executor calls `netbox_connector.get_next_available_ip(vlan_id="prod-web-vlan")`. The connector returns `{"ip_address": "10.1.2.3"}`.
    * **Step 3 (Delegated Execution):** The Tool Executor now has all the necessary parameters. It makes a single call to the RAF: `raf.execute_workflow(name="provision_vm", ip_address="10.1.2.3", change_ticket="CHG00123", zone="production")`.
5.  **RAF Execution:**
    * The RAF receives the request and executes its internal, audited `provision_vm` script. This script contains the complex logic to interact with CloudStack, potentially update NetBox again, and perform other system configurations.
    * The RAF returns a final status to the RAA (e.g., `{"status": "SUCCESS", "vm_id": "vm-abc-123"}`).
6.  **RAA - Result Handling:**
    * The RAA consolidates the results of the entire plan.
    * It passes a comprehensive status update back to the SERA Orchestrator for presentation to the user (e.g., "VM vm-abc-123 has been successfully provisioned at 10.1.2.3 under change ticket CHG00123.").

---

## 4. LLM Interaction

* **Model Choice:** RAA will use the same LLM instances available to the SERA Orchestrator (e.g., Ollama with Gemma, NVIDIA NIM). The configuration may allow specifying model preferences or capabilities required for RAA's planning tasks.
* **Prompt Engineering:** RAA will maintain its own set of system prompts tailored for:
    * Understanding infrastructure automation intent.
    * Mapping requests to a sequence of Connector and/or RAF calls.
    * Generating parameters for tool calls.
    * Generating code/config to be sent to the RAF for AI-assisted tasks.
* **Context:** When calling the LLM for planning, the `LLMTaskFormulator` will provide rich context, including:
    * The user's original, unaltered request.
    * The full list of discovered capabilities from the `CapabilityManager` (all available Connector functions and RAF workflows).
    * Relevant conversation history to disambiguate the current request.
    * Templates or examples of RAF report code or playbook YAML for generation tasks.

---

## 5. Configuration & Dependencies

* **Configuration:**
    * RAA-specific settings will be part of SERA's main configuration files (e.g., `config/settings.yaml`).
    * This includes the RAA's tool scope tag, default LLM models/parameters for planning, and RAA-specific prompt templates.
* **Dependencies:**
    * gRPC client libraries and compiled Protobuf stubs for all Connectors and Frameworks it might interact with. These will be managed centrally by SERA.
    * LLM client libraries (shared with the SERA Orchestrator).
    * These dependencies will be part of SERA's main `requirements.txt` and managed by SERA's `bootstrap_helper.py`.

---

## 6. Logging & Error Handling

* **Logging:** RAA will use SERA's centralized logging mechanism. All logs generated by RAA should be clearly identifiable with a `[RAA]` prefix to distinguish them from the main orchestrator logs.
* **Error Handling:**
    * RAA is responsible for handling errors from its tool calls (gRPC errors, application-level errors from Connectors/RAF) and from its own LLM interactions.
    * It will interpret these errors and translate them into a format understandable by the SERA Orchestrator, which then decides on the final user-facing message or a potential recovery strategy.

---

## 7. Security Considerations

* RAA operates under the user context provided by the SERA Orchestrator.
* When calling Connectors or the RAF, RAA passes this user context along. Each endpoint is responsible for its own authorization based on that context.
* The primary security model is that RAA plans but does not execute sensitive logic directly. The execution is handled by either:
    * Specialized, single-purpose Connectors.
    * The secure, sandboxed RAF environment for complex workflows.
* Any AI-generated code is never executed by RAA; it is passed to the RAF as data to be handled by its secure registration and execution mechanisms.

---

## 8. Future Considerations

* **More Sophisticated Planning:** Evolve the `LLMTaskFormulator` to handle conditional logic, loops, and error recovery paths within its generated plans.
* **State Management:** For long-running, multi-turn interactions where a user refines an automation plan with RAA before execution, RAA might need simple state management, though this would ideally be handled by SERA's main conversation history manager.
* **Feedback Loop:** Implement a mechanism where RAA can learn from successful and failed plans. For example, if a sequence of Connector calls frequently succeeds, SERA could use this data to fine-tune a specialized planning model or create a new RAF workflow to encapsulate the pattern.