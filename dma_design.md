# /ai/assets/docs/tool_guides/eas_design.md
## DocuMind Assistant (DMA) - Design Document

**Version:** 0.1  
**Date:** 2025-07-09  
**Author(s):** Martin Kecka, Gemini AI
**Category:** Assistant (A)
**Acronym:** DMA (DocuMind Assistant)

This document outlines the initial design for the DocuMind Assistant, a specialized AI module responsible for generating comprehensive documentation for software applications, modules, or services. It will initially function as a standalone application, with a long-term goal of integration into the SERA Orchestrator.

---
## 1. Architecture Overview

The DocuMind Assistant will follow a similar architecture to the Exchange Admin Assistant, focusing on a robust backend for AI interaction and a user-friendly frontend.

* **Frontend (UI)**: A **Streamlit** application (`documind_assistant.py`) will provide an intuitive interface for users to submit requests for documentation, specify inputs (e.g., code snippets, existing design plans, features lists), and view the generated documentation.
* **LLM Client** (`llm_client.py`): This core module will interact with a configured Large Language Model (LLM) endpoint (e.g., Ollama or NVIDIA NIM). It will be responsible for:
    * Taking the user's documentation request and contextual information.
    * Formulating precise prompts for the LLM.
    * Receiving and parsing the LLM's generated documentation.
    * Handling multi-stage generation (e.g., outline first, then content).
* **Content Processors** (`content_processors/`): A new sub-directory will house specialized processors responsible for handling different types of input content (e.g., code parsers, Markdown formatters, YAML/JSON readers). These will prepare input for the LLM and format output.
* **Configuration** (`config/settings.yaml` & `core/config_loader.py`): A centralized YAML file will store LLM endpoint details, any specific formatting preferences, and potential API keys for external services (if DocuMind later needs to pull info from Git, Jira, etc.).
* **Prompts** (`prompts/`): This directory will contain various prompt templates tailored for different documentation tasks (e.g., design plan generation, API documentation, user guides, code comments).
* **Data Store (Optional for Phase 1)**: For basic standalone operation, persistent storage might not be strictly necessary initially. However, for features like saving user preferences, generated document versions, or learning from feedback, a lightweight database (e.g., SQLite or a file-based JSON store) could be considered. For SERA integration, it would leverage SERA's VDB/PostgreSQL.

---
## 2. Project Structure

The DocuMind Assistant will mirror the clear separation of concerns seen in the Exchange Admin Assistant and align with the SERA Orchestrator's module expectations.

/documind_assistant/
├── documind_assistant.py           # Main application entry point (Streamlit UI)
├── bootstrap_helper.py             # For venv creation and dependency installation
├── requirements.txt                # Python dependencies
│
├── core/
│   ├── init.py
│   ├── llm_client.py               # Handles all LLM communication for documentation generation
│   ├── config_loader.py            # Loads settings from settings.yaml
│   └── document_generator.py       # Orchestrates documentation generation steps
│
├── content_processors/             # New: Modules for processing content
│   ├── init.py
│   ├── code_analyzer.py            # Analyzes code syntax/structure
│   ├── markdown_formatter.py       # Formats output into desired Markdown
│   └── yaml_parser.py              # Parses YAML structures (e.g., tool manifests)
│
├── utils/
│   ├── init.py
│   └── logging_setup.py            # Reusable logging configuration
│
├── config/
│   ├── init.py
│   └── settings.yaml               # Application configuration
│
├── prompts/
│   ├── init.py
│   ├── documentation_planner_prompt.py # High-level planning for documentation structure
│   ├── api_doc_prompt.py           # For generating API documentation
│   ├── design_doc_prompt.py        # For generating design documents
│   └── code_comment_prompt.py      # For generating code comments/explanations
│
├── docs/
│   └── documind_assistant.md       # This documentation file
│
└── tests/ (Future)
└── test_doc_generation.py


---
## 3. Core Features & Responsibilities

* **Natural Language Request Processing**: Understands user requests like "Generate a design plan for a new module that does X" or "Write API documentation for this Python function."
* **Multi-Format Input Support**: Ability to ingest various inputs for documentation, such as:
    * Raw text descriptions.
    * Code snippets (Python, YAML, JSON, etc.).
    * Existing Markdown or text files.
* **Structured Documentation Generation**: Produces documentation in a structured format, primarily Markdown, ensuring headings, code blocks, lists, and tables are correctly rendered.
* **Modular Prompting**: Leverages different prompt templates for various documentation types to ensure highly relevant and accurate output.
* **LLM Integration**: Seamlessly communicates with a configured LLM to perform the generative tasks.
* **Error Handling & Feedback**: Provides clear feedback to the user if the documentation generation fails or if additional information is required.
* **Interactive Refinement (Future)**: Allow users to provide feedback or request revisions on generated documentation within the UI.

---
## 4. Interaction Flow (Phase 1: Standalone)

1.  The user launches the Streamlit application.
2.  The user inputs a request (e.g., "Generate a design document for the DocuMind Assistant based on its purpose and proposed structure.") and provides any relevant input (e.g., copies the project structure above into a text area).
3.  The `llm_client.py` prepares the prompt using the appropriate template from `prompts/`.
4.  The `llm_client.py` sends the prompt and input content to the configured LLM.
5.  The LLM processes the request and generates the documentation content.
6.  The `content_processors/markdown_formatter.py` ensures the output is well-formatted Markdown.
7.  The UI (`documind_assistant.py`) displays the generated documentation to the user.
8.  The user can copy the documentation or refine their request.

---
## 5. Local Development & Setup

* **LLM Endpoint**: Similar to the Exchange Admin Assistant, the `config/settings.yaml` file must point to an accessible LLM endpoint (e.g., Ollama or NVIDIA NIM).
* **Local Python Environment**: Standard Python setup with virtual environment. `bootstrap_helper.py` will facilitate this.
* **Dependencies**: `requirements.txt` will include `streamlit`, `requests`, `pyyaml`, and any libraries needed for content parsing (e.g., `tree-sitter` for code analysis, `jsonschema` for validating structured inputs).
* **Launch the application from your terminal**:

    ```bash
    python3 documind_assistant.py
    ```

---
## 6. Future Integration with SERA Orchestrator

The DocuMind Assistant is designed for future integration as an external tool within the SERA Orchestrator. Its tool definition in `config/tools.d/` would resemble:

```yaml
# /ai/sera_orchestrator/config/tools.d/documind_assistant.yaml
- name: "documind_assistant"
  description: "Generates various types of documentation (design plans, API docs, code comments) based on provided content and requirements."
  parameters:
    type: "object"
    properties:
      document_type:
        type: "string"
        description: "The type of documentation to generate (e.g., 'design_plan', 'api_documentation', 'code_comments', 'user_guide')."
        enum: ["design_plan", "api_documentation", "code_comments", "user_guide"] # Example types
      input_content:
        type: "string"
        description: "The raw content or context for which to generate documentation (e.g., code, text, YAML)."
      requirements:
        type: "string"
        description: "Specific instructions or requirements for the documentation (e.g., 'focus on security aspects', 'include a table of contents')."
    required: ["document_type", "input_content"]
  llm_instruction_guide_path: "/ai/documind_assistant/docs/documind_assistant_guide.md" # Relative path within shared assets
  execution:
    type: "api"
    base_api_url: "grpc://localhost:500XX" # Placeholder for DocuMind's gRPC service port
    grpc_service_config:
      stub_directory_path_from_assets: "documind_assistant_protos/generated"
      stub_module_name: "documind_api"
      service_name_in_proto: "DocuMindService"
      rpc_method_name: "GenerateDocumentation"
      request_message_name: "GenerateDocumentationRequest"
  version: "0.1"
```

## 6. High-Level Plan & TODOs (Phase 1 Focus)

### Phase 1: Standalone MVP

* [ ] **Name Selection**: Confirm "DocuMind Assistant" or choose another suitable name.
* [ ] **Project Scaffolding**: Create the outlined directory structure.
* [ ] `bootstrap_helper.py` & `requirements.txt`: Implement basic bootstrap and define initial dependencies.
* [ ] `config/settings.yaml`: Define configuration structure (LLM endpoints).
* [ ] `core/config_loader.py`: Implement configuration loading.
* [ ] `utils/logging_setup.py`: Implement application logging.
* [ ] `prompts/`:
    * Create `documentation_planner_prompt.py` (for high-level structuring of docs).
    * Create initial prompt templates for common doc types (e.g., `design_doc_prompt.py`, `api_doc_prompt.py`).
* [ ] `core/llm_client.py`: Implement LLM communication (similar to Exchange Admin Assistant's `_post_request` and JSON parsing).
* [ ] `content_processors/`:
    * Implement basic `markdown_formatter.py` (ensuring correct Markdown output).
    * Implement `yaml_parser.py` to parse structured inputs (like the tool manifest or design plans).
    * (Optional for MVP) `code_analyzer.py` for basic code parsing.
* [ ] `core/document_generator.py`: This module will orchestrate the call to the LLM Client and apply any post-processing via `content_processors`. It will be the main logic unit for "generating documentation."
* [ ] `documind_assistant.py` (Streamlit UI):
    * Basic input text area for requests and content.
    * Display area for generated documentation.
    * "Generate" button to trigger the process.
    * (Optional) Basic selection for documentation type (which influences prompt used).

### Phase 2: Refinement & Advanced Features (Standalone)

* [ ] **Confluence Integration**:
    * [ ] Implement `integrations/confluence_api.py` for reading from and writing to Confluence (requires API key/token setup in `settings.yaml`).
    * [ ] Update `core/document_generator.py` to optionally use `confluence_api.py` for input retrieval and output publishing.
    * [ ] Enhance UI to allow specifying Confluence page IDs/URLs for input and output.
* [ ] **Documentation Update & Review**:
    * [ ] Create `prompts/documentation_update_prompt.py` to instruct the LLM on how to compare existing documentation with new content and propose updates.
    * [ ] Implement logic in `core/document_generator.py` to:
        * Accept existing documentation content and new/changed content as input.
        * Formulate a prompt that enables the LLM to identify differences and suggest updates.
        * Process the LLM's output to present changes (e.g., as a diff or highlighted sections).
    * [ ] Develop or integrate a lightweight diffing utility (if not handled entirely by the LLM).
* [ ] **User Feedback & Iteration**: Implement UI elements for users to provide feedback and trigger re-generation or refinement.
* [ ] **Version Control Integration**: (Future) Ability to pull content from Git repositories or push generated documentation.
* [ ] **Structured Data Input**: Enhance `content_processors` to handle more complex structured data (e.g., OpenAPI specs, existing design documents in JSON/YAML).
* [ ] **Learning from Feedback**: (Advanced) Potentially store successful prompts/outputs or user refinements to improve future generations.

### Phase 3: SERA Orchestrator Integration

* [ ] **gRPC API Definition**: Define `.proto` files for `DocuMindService` (e.g., `GenerateDocumentation` RPC) and its request/response messages.
* [ ] **gRPC Server Implementation**: Implement the gRPC server within `documind_assistant.py` or a dedicated `grpc_api/server.py` within the DocuMind module.
* [ ] **Stub Generation & Placement**: Document and automate the process of generating Python gRPC stubs and placing them in the SERA Orchestrator's shared protos asset directory.
* [ ] **SERA `tools.d/` Definition**: Create the `documind_assistant.yaml` file in SERA's tool manifest directory with accurate `grpc_service_config`.
* [ ] **End-to-End Testing**: Test calling DocuMind Assistant from the SERA Orchestrator.