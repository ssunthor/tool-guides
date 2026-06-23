# Bank Statement Ingestor (BSI) - Design Document

**Version:** 0.5.0
**Date:** 2026-02-21
**Author:** Martin Kecka & Gemini AI
**Category:** Ingestor (I)
**Acronym:** BSI

---

## 1. Introduction & Purpose

The **Bank Statement Ingestor (BSI)** is a dedicated backend service that automates the ingestion, parsing, and standardization of bank statement files. It is the primary source of truth for incoming cash, providing the foundational transaction data that the **Cash Application Assistant (CAA)** relies on to perform payment matching.

The BSI's core purpose is to process statement files from various banks, each with its unique format (e.g., CSV, XLSX). It handles files received via two primary channels: a **gRPC API** for programmatic uploads and a **local directory scanner** for batch processing.

Upon ingesting a file, BSI:
1.  Identifies the source bank (e.g., Deutsche Bank, Bank of America).
2.  Uses a bank-specific converter to parse the file, extracting all individual transactions and account balances.
3.  **Enriches** the data by querying the shared PostgreSQL database to:
    * Identify and attach the correct branch name (via `BranchHelper`).
    * Extract structured invoice and credit memo numbers from unstructured transaction text (via `InvoiceParserHelper`).
4.  Saves the standardized, enriched data into a set of dedicated PostgreSQL tables (`bsi_statement_files`, `bsi_transactions`, `daily_account_balances`).

This process transforms raw, bank-specific files into a clean, unified, and queryable dataset, ready for consumption by the downstream CAA module.

---

## 2. Architecture & Core Components

The BSI is a self-contained Python application built with a clear, modular architecture.

* `bank_statement_ingestor.py`: The main entry point that loads configuration, sets up logging, and launches the command-line interface (CLI).
* `cli/bsi_cli.py`: The `click`-based CLI. This is the primary user-facing tool for managing the service, providing commands to `init-db`, `start-grpc-server`, `upload-statement`, `view-statements`, `reprocess-statement`, and launching the continuous `start-poller`. It also includes nested groups for `analysis` and `dev` tools.
* `core/orchestrator.py`: The heart of the processing logic. It manages the processing of a single file (`process_single_file`) by selecting the correct converter based on a dynamic `PARSER_REGISTRY`. It also contains the logic for scanning the local input directory (`process_local_directory`).
* `core/ingestion_manager.py`: Manages the continuous, automated polling of remote SFTP servers. It scans for new files, applies lookback filters, syncs related addenda files, triggers the orchestrator, and archives processed files to the vault.
* `core/services.py`: The business logic layer that sits between the interfaces (CLI/gRPC/Poller) and the orchestrator.
    * `process_manual_upload`: Handles the full lifecycle of a manual file upload (local or gRPC), including syncing addenda, bank detection, orchestration, archiving, and triggering database writes.
    * `reprocess_statement_by_file_id`: Implements the complex logic for reprocessing, which involves finding the original file in the archive based on DB data, re-parsing it, and updating the database records.
    * `reapply_invoice_parsing_rules`: A database-only operation that re-extracts invoice candidates from raw text without requiring a full file re-parse.
* `core/converters/` & `core/parsers/`: Directories of bank-specific parsers and data structure converters.
    * `deutsche_bank`: Parses the specific MT940 format from Deutsche Bank.
    * `bank_of_america`: Parses BAI2 (US and EMEA) and custom XLSX formats from Bank of America.
    * `danske_bank`: Parses specific MT940 format from Danske Bank.
    * `itau_bank`: Parses specific MT940 format from Itaú Unibanco.
    * `ubb_bank`: Parses specific MT940 format from UBB.
    * `jsc_otp_bank`: Parses specific MT940 format from JSC OTP.
* `core/enrichers/`: Modules that augment raw parsed data.
    * `bofa_enricher.py`: Ingests auxiliary `.dat` addenda files (Lockbox, ACH, Wire) to inject highly accurate Payer Names, Account Numbers, and Invoice matches into standard BoA transactions.
* `core/analysis/`: A suite of deep-dive QA and development tools.
    * `invoice_qa.py`, `payer_qa.py`, `transaction_code_analyzer.py`, `rule_validator.py`, `local_parser.py`, `ingestion_stats.py`.
* `core/connectors/`: Interfaces for external system communication.
    * `cmd_connector.py`: A gRPC client that connects to the central Cash Management Database (CMD) to validate extracted invoice numbers and fetch rich metadata (e.g., Client Names for missing payers).
    * `sftp_connector.py`: A robust wrapper around `paramiko` for secure file discovery and downloading.
* `grpc_api/`: Defines and implements the external gRPC interface.
    * `protos/bsi.proto`: The Protobuf definition file.
    * `services.py` & `generate_stubs.py`: Auto-generates stubs on startup and hosts the Python implementation of the gRPC server.
* **Data Layer (Shared DB):**
    * `core/database/`: Contains `database.py` (engine/session management) and `models.py` (SQLAlchemy ORM definitions including `bsi_statement_files`, `bsi_transactions`, `daily_account_balances`, and linkages to `physical_bank_accounts`).
    * `core/io_handlers/`: Modules for all database interaction. `sql_writer.py` handles saving all parsed data (including smart merging and orphan deletion during reprocessing), and `sql_reader.py` handles data retrieval.
* **Enrichment Helpers (DB-Driven & CMD-Driven):**
    * `core/branch_helper.py`: A singleton class that resolves human-readable branch names using static mappings and database lookups.
    * `core/invoice_parser_helper.py`: A singleton class that queries `invoice_rules` to fetch correct regex patterns, resolving complex split/truncated invoice text into clean candidates.
    * `core/invoice_validator.py`: Takes candidates found by the parser helper, calls the `cmd_connector` to verify their existence in the financial system, prunes invalid "noise", and backfills missing Payer Names based on validated invoice metadata.
* `config/settings.yaml`: The central configuration file managing DB, gRPC, SFTP, Polling intervals, and dynamic Bank Profiles.
* `scripts/map_management.py`: *(Deprecated)* Legacy utility scripts, now fully replaced by live database rules and the `core/analysis` suite.

---

## 3. How a Bank Statement is Processed

The BSI processes files via four main workflows: gRPC upload, Local Directory scanning, Reprocessing, and Continuous SFTP Polling.

### Workflow 1: gRPC Upload (Real-Time)
1.  **Ingestion:** The `grpc_api/services.py` server receives a request containing the `filename` and raw `file_content` (bytes), saving it to a temporary location.
2.  **Addenda Sync (Conditional):** If detected as a Bank of America format, `services.process_manual_upload` connects via SFTP to download any matching `.dat` addenda files.
3.  **Bank Detection:** Sniffs the file content to identify the bank format (e.g., MT940, BAI2, XLSX).
4.  **Orchestration:** `core/orchestrator.process_single_file` calls the correct converter.
5.  **Conversion & Pre-Enrichment:** The converter parses the file into transactions. For BoA, the `BofaEnricher` merges the `.dat` addenda into the transactions. `InvoiceParserHelper` extracts potential invoice candidates, and `BankBranchHelper` resolves the branch.
6.  **Archiving:** The file (and any supporting addenda) is copied to a permanent, date-stamped vault location.
7.  **Database Write:** Structured data is passed to `sql_writer.save_statement_data` to create/update the statement header, insert transactions, and update daily balances.
8.  **Validation & Pruning:** `invoice_validator.validate_and_prune` runs asynchronously. It batches all extracted invoice candidates, sends them to the `cmd_connector`, prunes invalid matches (noise), and enriches the `payer_name` based on the validated invoice client data.

### Workflow 2: Local Directory Scan (Batch/Dev)
1.  **Trigger:** `cli/bsi_cli.py` calls `core/orchestrator.process_local_directory`.
2.  **Scan:** Walks the nested directory structure defined in `paths.local_input_dir`.
3.  **Process:** For each file found, executes Steps 4, 5, 7, and 8 from the gRPC workflow using the directory name as the `bank_name`.

### Workflow 3: Reprocessing
1.  **Trigger:** `reprocess-statement --file-id <UUID>` CLI command is executed.
2.  **Lookup:** Queries `bsi_statement_files` to find the bank name, date, and original vaulted filename.
3.  **Re-Parse:** Opens the *archived* vault file and reparses it entirely using the updated converter logic.
4.  **Smart Merge:** Calls `sql_writer.save_statement_data` in **reprocess mode**. This preserves existing transaction UUIDs where possible, deletes orphaned records, inserts new ones, and recalculates integrity checks.
5.  **Re-Validate:** Triggers the `invoice_validator` to verify any newly extracted invoices against the CMD.

### Workflow 4: Continuous SFTP Polling (Automated Ingestion)
1.  **Trigger:** The `start-poller` CLI command initiates the `IngestionManager`.
2.  **Interval Management:** Sleeps based on `poller.interval_seconds`, using tighter intervals during peak "Rush Hours".
3.  **Remote Scan:** Connects to bank-specific SFTP paths defined in `settings.yaml` (`bank_profiles`). Lists files, applying a lookback window (e.g., last 7 days) and regex filtering.
4.  **Delta Check:** Queries the database (`sql_reader.get_existing_filenames_global`) to filter out files that have already been ingested.
5.  **Download & Process:** Downloads new files (and associated addenda) to the temp directory. Executes the orchestration, writing, archiving, and validation steps identical to Workflow 1.

---

## 4. Key Features & Functionality

* **Continuous SFTP Poller**: An automated background worker (`IngestionManager`) that actively monitors remote bank servers, dynamically adjusting its polling interval based on peak business hours, and strictly ensuring no duplicate files are ingested.
* **Intelligent Addenda Merging**: Specifically for complex banks (like Bank of America), BSI downloads raw `.dat` lockbox/wire addenda files and perfectly reconciles them against summary transactions in the main statement, exploding them into individual line items with rich payer metadata.
* **CMD Validation & Noise Pruning**: Extracted invoice candidates are not trusted blindly. BSI communicates with the central Cash Management Database (CMD) via gRPC to validate every candidate. Invalid regex matches are automatically pruned, guaranteeing 100% downstream accuracy for the CAA.
* **Payer Reverse-Enrichment**: If a bank transaction lacks a payer name but contains a valid invoice number, BSI uses the `cmd_connector` to reverse-lookup the invoice owner and dynamically backfills the missing `payer_name` field.
* **gRPC API & Auto-Stubs**: Provides endpoints for programmatic file uploads. The server automatically checks and compiles Protobuf stubs on startup to ensure interface synchronization.
* **Deep Analysis & QA Suite**: A powerful set of CLI tools (`analysis` group) that allows developers to run safe, local simulations against production data. This includes rule validation, transaction code gap analysis, payer extraction audits, and noise/pruning reports.
* **Modular Bank Converters**: A clean `core/converters` architecture supports Deutsche Bank, Bank of America (US/EMEA/XLSX), Danske Bank, Itaú Unibanco, UBB, and JSC OTP.
* **Robust Reprocessing & Delta Merging**: Allows developers to update parsing logic and run the reprocessing command. The system automatically preserves transaction IDs, deletes orphans, and safely updates data without breaking downstream CAA linkages.

---

## 5. Data Storage

### 5.1. PostgreSQL Database (Shared)

The BSI module both writes to its own tables and reads from shared CAA tables within the same database.

* **Tables Owned/Written by BSI:**
    * `bsi_statement_files`: Stores one record per account per processed file. Contains summary/header info (balances, dates, account number) and the `file_id` (UUID).
    * `bsi_transactions`: Stores all individual transaction lines. Linked to `bsi_statement_files` by `file_id`.
    * `daily_account_balances`: Stores opening and closing balances, with a unique constraint on `(account_number, currency, statement_date)`.

* **Tables Read by BSI (for Enrichment):**
    * `bank_accounts`: Read by `BranchHelper` to find `branch_name` from `bank_account_number`.
    * `business_units`: Read by `InvoiceParserHelper` as part of the join.
    * `invoice_rules`: Read by `InvoiceParserHelper` to get the `prefix` and `length` for invoice regex patterns.

### 5.2. File System (Storage)

* **Input Directory (Local Batch):** `/ai/bank_statement_ingestor/input/`
    * Processed files are expected to be pre-sorted: `.../<bank_name>/<YYYY>/<MM>/<file.csv>`
* **Archive Directory (gRPC Uploads):** `/ai/assets/<csv|xlsx>/bsi/`
    * Successfully uploaded files are archived here: `.../<bank_name>/<YYYY>/<MM>/<MM.DD.YYYY>.csv`
* **Temp Directory:** `/ai/assets/tmp/bank_statement_ingestor/`
    * Used for temporarily storing gRPC uploads during processing.

---

## 6. Configuration

The application's behavior is controlled by `config/settings.yaml`.

* **`grpc_server`**: Host, Port (`50052`), and max worker settings for the inbound upload API.
* **`cmd_connector`**: Host and Port (`50053`) for outgoing verification requests to the CMD system.
* **`database`**: The DSN connection string for the PostgreSQL database.
* **`sftp`**: Global connection credentials (host, port, username, RSA key path) for remote bank servers.
* **`poller`**: Controls the `IngestionManager` behavior (enabled status, interval seconds, lookback days).
* **`bank_profiles`**: Dynamic dictionary controlling which banks are active. Maps a `parser_id` to its specific `remote_path`, `addenda_path`, and `filename_pattern` regex.
* **`paths`**:
    * `local_input_dir`: Path for the batch processing command.
    * `temp_files_dir`: Transient location for downloads and uploads.
    * `vault_dir`: Permanent, date-partitioned archive for all successfully processed files.
    * `dev_tests_dir`: Root directory for local QA simulations and output reports.
    * `generated_stubs_path` & `cmc_generated_stubs_path`: Paths for auto-generated Protobuf files.
* **`logging`**: Log level, directory, and daily rotating filenames.

---

## 7. Command-Line Interface (CLI) Usage

The main entry point is `bank_statement_ingestor.py`. It utilizes a robust `click` interface with main commands and specialized sub-groups (`analysis`, `dev`).

```bash
# --- Core Services ---
# Initialize the database (drops and recreates BSI tables safely)
./bank_statement_ingestor.py init-db

# Start the inbound gRPC server to listen for uploads
./bank_statement_ingestor.py start-grpc-server

# Start the continuous SFTP background poller
./bank_statement_ingestor.py start-poller

# --- Manual Operations ---
# Manually upload and ingest a file (automatically determines bank/archiving)
./bank_statement_ingestor.py upload-statement --file "/path/to/statement.xlsx"

# Reprocess a specific statement file from the archive using its DB UUID
./bank_statement_ingestor.py reprocess-statement --file-id "a1b2c3d4-..."

# View ingested statements with granular filters
./bank_statement_ingestor.py view-statements --account "00039429000"
./bank_statement_ingestor.py view-statements --filename "11.04.2025"

# View recent ingestion events timeline and addenda linkage
./bank_statement_ingestor.py ingestion-stats --limit 20

# Deep inspection of a single transaction (dumps all columns and tests live extraction)
./bank_statement_ingestor.py inspect-tx --tx-id "b1c2..."

# --- QA & Analysis Suite (analysis group) ---
# Analyzes local files to find transactions where Payer Name extraction failed
./bank_statement_ingestor.py analysis payer-qa --bank "deutsche_bank" --direction CREDIT

# Analyzes transaction codes across local files to identify missing dictionary mappings
./bank_statement_ingestor.py analysis code-qa --unknown-only

# Simulates full extraction and CMD validation, generating a deep-dive Pruning/Noise report
./bank_statement_ingestor.py analysis qa-extraction --test-data

# Evaluates configuration coverage by matching DB accounts to local file contents
./bank_statement_ingestor.py analysis validate-rules --test-data

# --- Development Tools (dev group) ---
# Dry-run SFTP connection to list or download files without saving to DB
./bank_statement_ingestor.py dev test-sftp-parse deutsche_bank --latest --days 3

# Parses local files and generates detailed JSON outputs for manual inspection
./bank_statement_ingestor.py dev local-parse danske_bank
```

## 8. Future Enhancements

* **Activate Bank of America Converter**: The `bank_of_america_converter.py` is implemented, but it is currently commented out in the `core/orchestrator.CONVERTER_MAP`. The immediate next step is to test this converter with real files, ensure the `_detect_bank` service in `core/services.py` reliably identifies its specific XLSX format, and activate it in the map to support BoA files from gRPC uploads.

* **Database-Driven Bank Detection**: The current `_detect_bank` service in `core/services.py` uses hardcoded `if/elif` logic to identify bank formats based on unique column names. This is brittle and requires a code deployment to add new banks.
    * **Solution**: This logic should be migrated to the database. A new table (e.g., `bsi_bank_profiles`) could store detection rules, such as `{"file_type": "csv", "required_columns": ["Account No", "Bank/Branch"]}` and map them to a specific converter (e.g., "deutsche_bank"). The `_detect_bank` function would then query this table and loop through the profiles, making the system extensible without new code.

* **Tighter CAA Feedback Loop**: Currently, BSI is a "write-only" service; CAA consumes its data but cannot send feedback. This limits BSI's ability to learn or self-correct.
    * **Solution**: A new gRPC endpoint should be added to BSI, such as `ReparseTransactionInvoices(transaction_id)`. When the CAA `RuleEngine` applies a payment but finds that BSI failed to extract the invoice (i.e., `bsi_transaction.extracted_invoice_numbers` is empty), CAA could call this endpoint. The BSI service would then re-fetch that single transaction's `raw_bank_text` and re-run the `InvoiceParserHelper` on it, updating the `extracted_invoice_numbers` field. This is far more efficient than the current file-level `reprocess-statement` command.

* **Deprecate Static Map Utilities**: The CLI includes commands like `generate-invoice-map` and `analyze-account-map` which rely on static files (`company_details.csv`, `account_to_invoice_rules.json`). However, the live runtime logic (in `core/invoice_parser_helper.py`) correctly queries the `invoice_rules` table from the shared database.
    * **Solution**: These static utilities are misleading and are development relics. They should be removed to eliminate the "dual source of truth." A new utility, e.g., `analyze-db-rules`, could be created to perform a gap analysis by comparing local statement files against the *live database rules*, which would provide actual, actionable insights.