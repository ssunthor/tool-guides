# /ai/assets/docs/tool_guides/ema_design.md
## Email Assistant (EMA) - Design Document

**Version:** 0.5  
**Date:** 2026-03-16  
**Author(s):** Martin Kecka, Gemini AI
**Category:** Assistant (A) / Gateway
**Acronym:** EMA (Email Assistant)
**Microsoft Exchange Online email account:** `sera-ai@gtt.net`

## 1. Introduction

The EmailTool module is an AI-powered assistant named "Sera - SysEng Reasoning Assistant (AI)." Its primary function is to monitor a designated Microsoft Exchange Online email account via the Microsoft Graph API. It processes incoming emails by leveraging a Large Language Model (LLM) to understand queries and generate helpful responses, and then sends these replies back through the same email account.

This module is designed to act as an intelligent virtual assistant, capable of answering questions and providing information based on the content of the emails it receives and its general knowledge from the LLM.

This is currently operating as a prototype standalone module. The functionality described herein is actively being refactored to serve as the dedicated **Email Gateway and Delegation Assistant** within the broader SERA Orchestrator architecture.

## 2. Features

* **Email Monitoring:** Continuously polls a specified Exchange Online mailbox for new unread emails.
* **Intelligent Processing:** Determines which emails to process based on direct addressing or a specific mention trigger ("@SERA AI") in group emails.
* **LLM-Powered Responses:** Utilizes a Large Language Model to understand email content and generate relevant, helpful replies.
* **Contextual Replies:**
    * Handles general queries based on email content.
    * Provides a self-introduction if specifically asked (e.g., "who are you?", "introduce yourself").
* **Automated Email Replies:** Sends replies using the Microsoft Graph API, properly threading them with the original email conversation.
* **HTML Email Parsing:** Parses incoming HTML emails to extract plain text content for LLM processing, with a heuristic to attempt to isolate the latest reply in a thread.
* **Caching:** Caches details of incoming and outgoing emails as JSON files for logging, debugging, and potential future analysis.
* **Configurable:** Key parameters such as API credentials, LLM model, polling intervals, and monitored email address are configurable.
* **Robust Authentication:** Uses MSAL (Microsoft Authentication Library) for secure OAuth2 client credentials flow authentication with the Microsoft Graph API.
* **Dynamic LLM Context:** Adjusts the context window (`num_ctx`) for LLM calls based on the estimated size of the input prompt.

## 3. Technical Architecture

### 3.1. Core Modules & Scripts

The EmailTool module is composed of several Python scripts primarily located under the `/ai/email_tool/core/` and `/ai/email_tool/prompts/` directories:

* **`main_email_tool.py`:**
    * The main executable script that orchestrates the email processing loop.
    * Handles command-line arguments (e.g., `--debug`).
    * Initializes logging and configuration.
    * Contains the primary loop for fetching, processing, and replying to emails.
    * Determines whether an email should be processed (direct address or mention trigger).
    * Handles introduction requests by loading its own documentation (`email_tool.md`) to provide context to the LLM.
* **`core/config.py`:**
    * Centralized configuration for the module.
    * Stores settings for MS Graph API credentials, LLM parameters, email account details, logging paths, and cache directories.
    * Includes functions to ensure necessary directories exist and to check for essential credentials.
* **`core/email_tool_utils.py`:**
    * Handles all interactions with the Microsoft Graph API.
    * Manages MSAL authentication and token acquisition.
    * Provides functions for fetching unread emails, marking emails as read, and sending replies.
    * Includes logic for parsing email messages (Graph API format to internal `EmailContent` model), including HTML to plain text conversion using `BeautifulSoup` and a custom heuristic `_extract_latest_reply_from_html` to isolate new content in replies.
* **`core/llm_interface.py`:**
    * Manages communication with the Large Language Model (LLM) via Langchain.
    * Initializes the `ChatOllama` client.
    * Formats prompts using `PromptTemplate`.
    * Calculates and binds dynamic context window sizes (`num_ctx`) for Ollama.
    * Sends requests to the LLM and retrieves responses.
* **`core/cache_manager.py`:**
    * Provides simple file-based caching functionality.
    * Used by `main_email_tool.py` to save `EmailContent` objects (both incoming and outgoing emails) as individual JSON files in the configured cache directory. *Note: Other caching functions within this file for per-file analysis or documentation caches are part of a broader analyst framework and not directly used by the email tool's primary email processing workflow.*
* **`core/models.py`:**
    * Defines Pydantic models, primarily `EmailContent`, which standardizes the structure for representing email data within the application.
* **`prompts/email_tool_prompts.py`:**
    * Contains pre-defined prompt templates for the LLM.
    * `EMAIL_REPLY_PROMPT`: Guides the LLM on how to formulate general replies, emphasizing a helpful persona, professional email formatting, and clear communication.
    * `INTRODUCTION_PROMPT_TEMPLATE`: Used when the AI is asked to introduce itself. It instructs the LLM to base its introduction on the content of this `email_tool.md` document.
* **`bootstrap_helper.py` (located at `/ai/email_tool/bootstrap_helper.py`):**
    * A utility script executed at the start of `main_email_tool.py`.
    * Ensures a Python virtual environment (`venv`) exists and is activated.
    * Installs required dependencies from `requirements.txt`.
    * Can attempt to install the `pythonX.Y-venv` system package on Debian/Ubuntu if missing.
    * Re-executes the main script from within the venv if necessary.

### 3.2. Configuration

Configuration is managed through `core/config.py`. Key settings include:

* **MS Graph API Credentials:**
    * `MSGRAPH_TENANT_ID`
    * `MSGRAPH_CLIENT_ID`
    * `MSGRAPH_CLIENT_SECRET`
    * These are hardcoded in `config.py` for development and MUST be correctly populated for the tool to function.
* **Email Monitoring:**
    * `EMAIL_ADDRESS_TO_MONITOR`: The email address of the mailbox Sera AI will monitor and send replies from.
    * `EMAIL_POLLING_INTERVAL_SECONDS`: How often to check for new emails.
    * `EMAIL_FETCH_LIMIT`: Maximum number of emails to fetch per poll.
* **LLM Settings:**
    * `DEFAULT_ANALYST_MODEL`: Specifies the LLM model to use (e.g., "gemma3:27b").
    * `MODEL_MAX_CTX`: Maximum context window supported by the model.
    * `CONTEXT_BUFFER_TOKENS`: Buffer tokens to reserve for the LLM's response.
* **Paths:**
    * `LOG_DIR`: Directory for log files.
    * `CACHE_DIR`: Directory for cached email data.
* **Debug Mode:**
    * `DEBUG_ENABLED`: Can be set via the `EMAIL_TOOL_DEBUG` environment variable or the `--debug` command-line flag.

### 3.3. Authentication (Microsoft Graph API)

* Authentication with the Microsoft Graph API is handled using the OAuth 2.0 client credentials grant flow.
* The `msal` (Microsoft Authentication Library for Python) package is used.
* `_initialize_msal_app()` in `email_tool_utils.py` sets up the `ConfidentialClientApplication`.
* `get_graph_api_token()` acquires an access token. Acquired tokens are cached in memory with their expiry time to minimize redundant requests.
* The required permissions (scopes) are defined by `MSGRAPH_SCOPES = ["https://graph.microsoft.com/.default"]`, which typically implies application permissions have been granted to the Azure AD App Registration corresponding to the `MSGRAPH_CLIENT_ID`.

### 3.4. Email Fetching & Parsing

1.  **Fetching:** Unread emails are fetched from the Inbox of the `EMAIL_ADDRESS_TO_MONITOR` using the `Workspace_unread_emails_graph()` function. This function filters for `isRead eq false` and orders by `receivedDateTime` to process older emails first. It selects specific fields to reduce payload size.
2.  **Parsing:** Each fetched Graph API message is parsed by `parse_graph_message()`:
    * Relevant fields (ID, sender, recipients, subject, received time, body content type) are extracted.
    * If the body is HTML (`contentType: 'html'`), `BeautifulSoup` is used to convert it to plain text. The function `_extract_latest_reply_from_html` attempts a heuristic parse to isolate only the most recent message in an email thread by looking for common reply separators (e.g., `<hr id="stopSpelling">`, "From:" blocks).
    * If the body is plain text (`contentType: 'text'`), it's used directly, with a basic heuristic to try and split off quoted replies.
    * The parsed data is structured into an `EmailContent` object.

### 3.5. Processing Logic (Triggers & Introduction)

For each incoming email, `main_email_tool.py` decides if and how to process it:

1.  **Sole Recipient Check:** If the `EMAIL_ADDRESS_TO_MONITOR` is the *only* recipient (To, Cc, Bcc), the email is processed.
2.  **Mention Trigger Check:** If the email is sent to multiple recipients (i.e., a group email where Sera AI's monitored address is one of the recipients), the email body (plain text) is scanned for the `MENTION_TRIGGER` (case-insensitive, default: "@SERA AI"). If found, the email is processed. Otherwise, it's marked as read and skipped.
3.  **Introduction Request:** If an email is flagged for processing, its body is checked for keywords defined in `INTRODUCTION_KEYWORDS` (e.g., "who are you", "introduce yourself").
    * If an introduction keyword is found, the `INTRODUCTION_PROMPT_TEMPLATE` is used. This prompt is given the content of this `email_tool.md` document as context, instructing the LLM to introduce Sera AI based on this documentation.
    * Otherwise, the `EMAIL_REPLY_PROMPT` is used for a general query.

### 3.6. LLM Interaction

1.  **Prompt Preparation:** Based on whether it's an introduction or a general query, the appropriate prompt template from `email_tool_prompts.py` is selected. Variables (sender email, subject, body, or introduction documentation) are filled into the template.
2.  **Context Sizing:** `llm_interface.py` estimates the token count of the formatted prompt. It then calculates a `num_ctx` value for the Ollama model, ensuring it's within the model's `MODEL_MAX_CTX` and includes a `CONTEXT_BUFFER_TOKENS` for the response. This `num_ctx` is bound to the LLM call.
3.  **LLM Call:** The `get_llm_response()` function sends the request to the Ollama LLM (configured via `DEFAULT_ANALYST_MODEL`).
4.  **Response Handling:** The LLM's plain text response is received.

### 3.7. Email Reply Generation

1.  The plain text response from the LLM is taken by `send_reply_graph()` in `email_tool_utils.py`.
2.  This text is converted into a simple HTML format:
    * Special HTML characters (`&`, `<`, `>`) are escaped.
    * Newline characters (`\n`) are converted to `<br>\n` tags to preserve line breaks in the email reply.
3.  The `/replyAll` endpoint of the Graph API is used, with the formatted HTML provided in the `comment` field of the request body. This allows the Graph API to construct the full threaded reply, including quoting the original message.
4.  The API call might directly send the email or create a draft which is then explicitly sent.

### 3.8. Caching

* The `cache_email_data()` function in `main_email_tool.py` uses `cache_manager.save_simple_cache()` to store `EmailContent` objects.
* Both incoming and outgoing emails are cached.
* Each email is saved as a separate JSON file in the directory specified by `config.CACHE_DIR`.
* Filenames are structured to be somewhat unique and identifiable, including direction (incoming/outgoing), timestamp, and a sanitized part of the message ID or subject.
* The cache includes the status of processing (e.g., `received_unprocessed`, `processing_llm`, `replied_direct`, `error_sending_reply`).

### 3.9. Logging

* Logging is configured in `main_email_tool.py` via the `setup_logging()` function.
* It uses Python's standard `logging` module.
* Logs are output to both the console (stderr) and a file (`email_tool.log`) located in `config.LOG_DIR`.
* Log level for the console can be set to INFO or DEBUG (via `--debug` flag or `EMAIL_TOOL_DEBUG` env var). File logging is always at DEBUG level.
* Log format includes timestamp, log level, module name, filename, line number, and the message.

## 4. Operational Guide

### 4.1. Prerequisites & Setup

1.  **Python Environment:** Python 3.x. The `bootstrap_helper.py` script aims to create a `venv` virtual environment.
2.  **Dependencies:** Required Python packages are listed in `requirements.txt`. `bootstrap_helper.py` will attempt to install these. Key dependencies include:
    * `msal` (for MS Graph API authentication)
    * `requests` (for HTTP requests to Graph API)
    * `beautifulsoup4` (for HTML parsing)
    * `langchain-community`, `langchain-core` (for LLM interaction)
    * `pydantic` (for data modeling)
3.  **Ollama & LLM Model:** An Ollama instance must be running and accessible to the EmailTool. The LLM model specified in `config.DEFAULT_ANALYST_MODEL` (e.g., `gemma3:27b`) must be downloaded and available in Ollama (`ollama pull gemma3:27b`).
4.  **Microsoft Azure AD App Registration:**
    * An Azure AD application registration is required.
    * The Application (Client) ID, Directory (Tenant) ID, and a Client Secret must be generated.
    * **Application Permissions** for Microsoft Graph API must be granted and admin consented. The scope `https://graph.microsoft.com/.default` implies that permissions like `Mail.ReadWrite` (to read emails and send replies from the monitored mailbox) and `Mail.Read` (potentially, depending on exact needs and if `Mail.ReadWrite` covers it for `isRead` updates) are necessary. Consult Microsoft documentation for the minimal required application permissions for reading, sending, and modifying read status of emails.
5.  **Configuration File:**
    * Copy or rename any template configuration to `core/config.py` (if applicable).
    * Edit `core/config.py` and accurately fill in:
        * `MSGRAPH_TENANT_ID`
        * `MSGRAPH_CLIENT_ID`
        * `MSGRAPH_CLIENT_SECRET`
        * `EMAIL_ADDRESS_TO_MONITOR` (the email address Sera AI will use)
        * Adjust `DEFAULT_ANALYST_MODEL` if using a different Ollama model.
        * Verify `LOG_DIR` and `CACHE_DIR` paths are writable.
6.  **File System:** Ensure the directories for logs (`LOG_DIR`) and cache (`CACHE_DIR`) specified in `config.py` are created and writable by the user running the EmailTool. The `ensure_directories()` function in `config.py` attempts to create these.

### 4.2. Managing the Service (systemd)

The EmailTool is designed to run as a `systemd` service (e.g., named `emailtool.service`) for continuous operation. Once the service unit file is created (e.g., in `/etc/systemd/system/`) and systemd is reloaded (`sudo systemctl daemon-reload`), you can manage it using standard `systemctl` commands:

* **Enable on boot:** `sudo systemctl enable emailtool.service`
* **Start:** `sudo systemctl start emailtool.service`
* **Stop:** `sudo systemctl stop emailtool.service`
* **Restart:** `sudo systemctl restart emailtool.service` (useful after code/config updates)
* **Check status:** `sudo systemctl status emailtool.service`
* **View logs:** Logs are typically sent to the systemd journal. Use `sudo journalctl -u emailtool.service` to view them, or `sudo journalctl -u emailtool.service -f` to follow new log entries.

The service will then run in the background, polling the configured email account.

### 4.3. User Interaction Guide

#### 4.3.1. How Sera AI Responds

Sera AI (the EmailTool persona) will process and potentially reply to emails under the following conditions:

1.  **Direct Emails:** If an email is sent directly *only* to the email address Sera AI monitors (defined in `config.EMAIL_ADDRESS_TO_MONITOR`), Sera will process it.
2.  **Group Emails with Mention:** If Sera AI's monitored email address is one of multiple recipients (e.g., part of a mailing list or a group conversation), Sera will *only* process the email if its body contains the specific trigger phrase:
    * **`@SERA AI`** (case-insensitive)

If an email is part of a group conversation and does *not* contain the "@SERA AI" trigger, Sera will ignore it (mark it as read and take no further action).

#### 4.3.2. Requesting an Introduction

If you want to know more about Sera AI's capabilities as an email assistant, you can ask it to introduce itself. Send an email (either directly or with the `@SERA AI` trigger in a group email) containing phrases like:

* "Who are you?"
* "Introduce yourself"
* "What can you do?"
* "Tell me about yourself"

Sera will then provide an introduction based on this documentation file.

### 4.4. Capabilities

As an email assistant, Sera AI's current capabilities are:

* **Monitoring a Mailbox:** It actively checks the configured email address for new messages.
* **Understanding Triggers:** It can differentiate between emails sent directly to it and emails where it's mentioned in a group discussion.
* **Basic Email Comprehension:** Using an LLM, it reads and understands the content of emails it's asked to process.
* **Generating Textual Replies:** It can formulate written responses to questions or comments in emails.
* **Self-Introduction:** It can explain its role and basic functions as this EmailTool module when asked.
* **Following Reply Guidelines:**
    * It strives to provide direct, actionable, and comprehensive answers.
    * It uses professional email language and formatting (paragraphs, natural flow).
    * It avoids explicit markdown in replies (e.g., no `**bold**` or `## headers`).
    * It greets the user and signs off as "Sera - SysEng AI Assistant".
* **Sending Replies:** It can send email replies using the monitored email account, threading them correctly.
* **Marking Emails as Read:** Processed emails are marked as read in the mailbox.

### 4.5. Current Limitations

It's important to understand what this EmailTool module, in its current form, **cannot** do:

* **No Access to External Tools or Systems (Natively):**
    * Sera AI, through this EmailTool module, **cannot directly interact with or query other systems** like NetBox, Zabbix, Jira, ServiceNow, Ansible, or any remote automation frameworks.
    * It cannot execute code or scripts.
    * It does not have access to the broader Orchestrator's tools or database capabilities (PostgreSQL, Vector Database for RAG) described in the "Orchestrator Architecture & Development Guide." Its knowledge is based on the LLM's pre-training and the content of the email it's processing.
* **No RAG (Retrieval Augmented Generation):** It does not have the ability to search external document repositories (e.g., Confluence, SharePoint) or vector databases to find specific information for its replies. Its answers come from its general LLM knowledge.
* **No Memory Beyond the Current Email (Limited History):** While it processes individual emails, it does not maintain a persistent, queryable conversation history across multiple separate email threads or over long periods in the way a chatbot with a dedicated history VDB might. (The caching is for individual transaction logging, not for contextual long-term memory for the LLM).
* **No Proactive Actions:** It only reacts to incoming emails that meet the processing criteria. It does not initiate emails or perform tasks autonomously.
* **Dependent on LLM Knowledge:** The quality and accuracy of its responses are dependent on the underlying LLM's training data and capabilities. It can sometimes provide incorrect or incomplete information.
* **No Understanding of User Identity/Permissions (Beyond Sender Email):** It treats all senders the same and does not have a concept of user roles, permissions, or access levels.
* **No File Attachment Processing:** It primarily processes the textual content of emails. It cannot analyze or understand file attachments.
* **Heuristic Reply Extraction:** The mechanism to extract the "latest reply" from an email thread is heuristic-based and might not be perfect for all email client formats or complex threads. It might sometimes include more of the email history in its context than just the very last message.
* **Limited to One Mailbox:** It can only monitor and operate from the single email address configured in `EMAIL_ADDRESS_TO_MONITOR`.
* **Limited to Dedicated People/Teams:** Trigger is limited to only SysEng Team and few other individuals.

### 4.6. Troubleshooting

* **Not Receiving Replies:**
    * **Check Logs:** Examine the log file in `config.LOG_DIR/email_tool.log` for errors. Enable debug logging (`--debug`) for more details.
    * **Credentials:** Ensure MS Graph API credentials in `config.py` are correct and the Azure AD App Registration has the necessary permissions (and admin consent). Check `check_email_credentials()` output in logs.
    * **Ollama Server:** Verify the Ollama server is running and the specified model is available.
    * **Network Connectivity:** Ensure the machine running EmailTool can reach the MS Graph API endpoints and the Ollama server.
    * **Trigger Phrase:** If it's a group email, ensure the `@SERA AI` trigger phrase was used correctly.
    * **Email Parsing Errors:** Check logs for issues related to parsing specific emails.
* **Incorrect Replies:**
    * **Prompt Engineering:** The quality of replies depends heavily on the prompts in `prompts/email_tool_prompts.py` and the LLM's capabilities.
    * **LLM Model:** The chosen LLM might not be suitable for all tasks.
* **Authentication Errors (in logs):**
    * Often indicate issues with `MSGRAPH_CLIENT_ID`, `MSGRAPH_TENANT_ID`, `MSGRAPH_CLIENT_SECRET`, or API permissions in Azure AD. Look for HTTP 401/403 errors related to token acquisition or API calls.
* **Tool Stops Running:**
    * Check logs for critical unhandled exceptions.
    * Resource issues (memory, CPU) on the machine running the tool.

---

## 5. Architectural Evolution: The "Email Gateway" Pattern

The original design envisioned the SERA Orchestrator polling the email account directly, relying on a stateless, monolithic script architecture. To align with the SERA Ecosystem microservice standards (as implemented in CAP, CAA, and the core Orchestrator), EMA is being refactored into an independent **Email Gateway and Delegation Assistant**.

EMA will act as the dedicated "brain" for the email domain. It handles the messy reality of Exchange Online, determines the user's intent, answers simple queries directly, and securely delegates complex, long-running tasks to other specialized ecosystem modules (e.g., CV Analysis Processor, Remote Automation Framework) via gRPC.

### 5.1. Persistent State & Audit Management (PostgreSQL)

To support delegating tasks to external modules that may take minutes or hours to complete (e.g., parsing a batch of CVs), EMA must abandon its legacy `while True` polling loop and file-based JSON caching.

* **Database Integration:** EMA will implement a PostgreSQL database (`ai_assistant_db`) connection using the SQLAlchemy ORM.
* **Stateful Tracking:** Incoming emails will be written to an `email_messages` table. If a task is delegated to an external module, the message state is flagged as `AWAITING_MODULE` instead of blocking the main thread.
* **Thread Awareness:** By natively storing the `conversationId` and parsed history in an `email_threads` table, EMA eliminates the need to perform expensive, rate-limited Graph API calls just to retrieve the context of a previous conversation.
* **Structured Audit Logging:** A `delegated_tasks_audit` table will maintain a strict paper trail of all outbound delegations. This captures the sender, the matched intent, the exact gRPC payload sent to downstream modules, and the eventual return code/status.

### 5.2. Asynchronous Processing & Context Abstraction

The synchronous `time.sleep()` polling loop is a massive bottleneck. EMA will adopt the worker pattern utilized across the ecosystem, paired with strict data encapsulation.

* **Decoupled Polling:** A lightweight asynchronous poller will fetch Graph API emails and place them into a Redis queue.
* **Worker Fleet:** Dedicated worker nodes (using Celery or Python asyncio background tasks within a FastAPI shell) will pick up messages from the queue for AI classification and delegation.
* **Processing Context Abstraction:** Before an email hits the AI or any routing logic, the worker bundles the raw Graph API message, the parsed clean text, the sender identity, and the recipient modes (e.g., `is_sole_recipient`) into a strongly typed `ProcessingContext` data class. This ensures a clean, uniform data handoff throughout the pipeline.

### 5.3. Dynamic Intent Registry & "Right-Sized" Models

EMA will transition away from the legacy, local Ollama integration and connect directly to the AI Factory via a standardized `vLLMClient`. It relies on a database-driven registry to dynamically route requests without requiring hardcoded logic updates.

* **The Intent Registry (PostgreSQL):** A dedicated `ema_intent_registry` table stores active ecosystem routes. Each record contains an `intent_slug` (e.g., `cv_submission`), a natural-language `llm_description`, the `target_module` gRPC endpoint, and an `allowed_groups` list.
* **Dynamic Intent Classification (`gemma3:4b`):** All incoming emails pass through this high-speed routing model. EMA queries the active registry and dynamically assembles the LLM prompt, instructing the model to classify the email based strictly on the registered descriptions. The model outputs a structured JSON directive (e.g., `{"intent": "cv_submission", "confidence_score": 0.95}`).
* **Fallback & Ambiguity:** If the model returns a confidence score below a configured threshold (e.g., 0.70) or hallucinates an unregistered intent, EMA defaults to the `general_query` route to prevent routing failures.
* **Reasoning & Drafting (`gemma3:27b`):** Only when the intent is classified as a `general_query` needing a complex local reply, or when synthesizing a final response from a completed delegated task, will EMA invoke the larger, resource-intensive reasoning model.

### 5.4. Security, Eligibility & Graceful Refusals

EMA serves as the first line of defense for the ecosystem. It must prevent unauthorized users from triggering downstream automation modules.

* **Dynamic Group Resolution:** For intents that define an `allowed_groups` constraint (e.g., a firewall change request restricted to the Network DL), EMA utilizes the Microsoft Graph API to dynamically resolve and expand Distribution Lists or AD Groups into a list of authorized email addresses. This data is cached temporarily to avoid API rate limits.
* **Graceful Degradation (Bounce Logic):** If the `gemma3:4b` model successfully classifies an intent, but the sender is *not* in the resolved `allowed_groups`, EMA blocks the delegation. Instead of failing silently or crashing, it immediately synthesizes a polite but firm "unauthorized request" email and sends it back to the user, closing the loop locally without ever waking up the target module.

### 5.5. Shared Vault Integration (Attachments)

Modules like CAP (CV processing) and LDP (Legal Documents) rely entirely on file attachments. 

* **Vault Storage:** EMA will be responsible for securely downloading attachments via the Graph API and saving them directly to the shared persistent volume at `/ai/assets/vault/ema/{thread_id}/`.
* **Pointer Passing:** When delegating tasks to other modules, EMA will pass the exact absolute file paths (pointers) in the gRPC payload, rather than attempting to stream binary data over the wire.

### 5.6. Bi-Directional gRPC Inter-Service Communication

To function as the ecosystem's communication hub, EMA will operate as both a gRPC Client and Server:

* **Outbound Delegation (Client):** When the `gemma3:4b` model classifies an email belonging to another domain (and it passes security checks), EMA translates the `ProcessingContext` and attachment pointers into a gRPC request and dispatches it to the target module's API.
* **Inbound Replier (Server):** EMA will expose an inbound gRPC service (e.g., `EmailGatewayService.SendReply`). When a delegated module finishes its long-running task, it calls this endpoint with the final synthesized text. EMA looks up the original `message_id` in Postgres, records the audit trail, and dispatches the final `replyAll` via the Graph API.

---

## 6. Implementation Plan & TODOs

The transition to the Email Gateway pattern will be executed in three phased updates to ensure stability during the refactor.

### Phase 1: Infrastructure & State Migration
- [ ] **Database Setup:** Define SQLAlchemy models (`EmailMessage`, `EmailThread`, `DelegatedTaskAudit`, `EmaIntentRegistry`) and configure the connection to the shared PostgreSQL instance.
- [ ] **Async Refactoring & Context:** Convert the synchronous polling loop into a Redis-backed queue worker architecture. Implement the `ProcessingContext` data class for clean pipeline data passing.
- [ ] **Vault Storage:** Implement logic to download and save email attachments directly to the shared `/ai/assets/vault/ema/` directory.

### Phase 2: Intent Engine & Security Routing
- [ ] **vLLM Client Integration:** Replace `langchain_community.chat_models.ChatOllama` with the standard ecosystem `vLLMClient`.
- [ ] **Dynamic Graph API Auth:** Implement the Microsoft Graph API expansion functions for Distribution Lists and AD Groups, complete with TTL caching.
- [ ] **Router & Refusal Logic:** Build the internal router that executes the `gemma3:4b` intent classification, validates the sender against the resolved `allowed_groups`, and triggers the localized "Bounce/Refusal" email logic if authorization fails.

### Phase 3: gRPC API & Cross-Module Delegation
- [ ] **Define Protobufs:** Create the `ema_service.proto` defining the `SendReply` and `SendNewEmail` RPCs for other ecosystem modules to consume.
- [ ] **Implement Inbound Server:** Stand up the gRPC server within the EMA application to listen for cross-module reply requests and execute the final Graph API `replyAll`.
- [ ] **Implement Outbound Clients:** Integrate dynamic gRPC stub loading (from `/ai/assets/protos/`) so EMA can construct and dispatch task payloads (with vault paths) to modules like CAP, CAA, or the Orchestrator.