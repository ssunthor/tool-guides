# CloudStack Assistant (CSA) - Quick Design Document
Here is a simple design document for the new assistant.

**Version:** 0.1  
**Category:** Assistant (A)  
**Acronym:** CSA

## 1. Introduction & Purpose
The CloudStack Assistant (CSA) is a temporary, special-purpose AI assistant designed to execute the second stage of the VM provisioning workflow. Its sole function is to translate a user's natural language request to deploy a specific VM into a precise command-line execution of the `netbox_cs.py` script.

It will bridge the gap between the NetBox data entry (done by the NBA) and the actual VM creation in CloudStack, providing a user-friendly interface to trigger the script, monitor its output, and receive a clear summary of the results.

## 2. Architecture & Integration
**Service Type:** Standalone Assistant Module with a Streamlit UI.

**Interacts with:**

* **Human User:** Via the `cloudstack_assistant.py` launcher, which starts the Streamlit web application.
* **LLM Service (AI Factory):** Used for two simple tasks: extracting entities from the user's query and summarizing the final script output.
* **Local Command-Line Tools:** Executes the `netbox_cs.py` script directly on the same server using Python's `subprocess` module. This requires that the script and its dependencies (like CloudMonkey) are installed locally.

## 3. Core Functionality & Process Flow ⚙️
### 3.1. Stage 1: Intent Recognition
The user's query (e.g., "Add the VM 'nyc-se-jh01' for ticket SCTASK0013607") is sent to the LLM. The LLM's only task is to extract two key entities: `vm_name` and `ticket_number`.

### 3.2. Stage 2: Command Generation
Using the extracted entities, the assistant constructs the exact shell command to be executed.

**Example:** `"/ai/cloudstack_assistant/tools/netbox_cs.py --add 'nyc-se-jh01' --ticket 'SCTASK0013607'"`

### 3.3. Stage 3: Local Execution
The assistant uses Python's `subprocess` library to execute the `netbox_cs.py` script directly on the local filesystem. It will capture the `stdout` and `stderr` streams from the script in real-time to display its progress directly in the UI.

## 4. User Interface (Streamlit)
The UI will be a simplified version of the NBA's interface:

* **Input:** A text area for the user's request and an "Execute" button.
* **Execution Log:** A dedicated area (e.g., a `st.expander` or `st.code` block) that shows the live output from the `netbox_cs.py` script as it runs.
* **Final Result:** A final summary box at the top (e.g., `st.success` or `st.error`) that displays the AI-synthesized summary of the outcome.

## 5. Configuration (config/settings.yaml)
The configuration file will be simplified to include:

* `llm_interface`: Connection details for the Ollama service.
* `logging` & `transaction_logging`: Standard logging configuration.
* `csa_config`: A new section containing parameters for the local execution.
    * `script_path`: "/ai/cloudstack_assistant/tools/netbox_cs.py"
    * `user_context`: Default user information.

## 6. Reusable Scripts from NetBox Assistant
You're right, we can reuse a significant portion of the NetBox Assistant (NBA) application structure for this new, simpler assistant. The NBC is a backend service and isn't relevant here. The goal is to create a new UI-driven assistant, so the NBA provides an excellent template.

Here are the files we will copy and adapt to create the CloudStack Assistant:

* ✅ `netbox_assistant.py`: We'll copy and rename this to `cloudstack_assistant.py`. It's the main launcher, and its logic for handling configuration, arguments, and starting the Streamlit app is perfectly reusable.
* ✅ `ui/nba_app.py`: This is the core of the UI. We'll copy and rename it to `ui/csa_app.py`. We will keep the overall Streamlit layout (input box, button, status/result display) but will drastically simplify the backend logic by removing the multi-stage pipeline and replacing it with a direct command execution flow.
* ✅ `core/llm_interface.py`: We still need to understand the user's natural language request (e.g., "deploy vm-web-01 for ticket SCTASK12345"). This client is essential for making the LLM call to extract the VM name and a ticket number.
* ✅ `utils/logging_setup.py` & `utils/transaction_logger.py`: Standardized logging and auditing are always valuable. We'll reuse these to maintain good observability, adapting the transaction logger to capture the user's request, the generated command, and the script's output.
* ✅ `config/settings.yaml`: We will use this as a template for the CSA's configuration, removing the `nbc_service_config` and adding new parameters for the remote script execution (e.g., server hostname, script path).
* ✅ `prompts/` directory: We will create two new, much simpler prompt files modeled on the existing ones:
    * A new intent recognition prompt to extract only the `vm_name` and `ticket_number`.
    * A new synthesis prompt to take the raw text output from the `netbox_cs.py` script and create a clean, human-readable summary.

The following files from the NBA will **not** be needed:

* ❌ `core/nbc_client.py`
* ❌ `core/planner.py`
* ❌ The more complex prompts like `plan_generation_prompt.py` and `failure_analysis_prompt.py`.