# Bank Statement Ingestor (BSI) - Design Document

**Version:** 0.1.0
**Date:** 2025-11-04
**Author(s):** Gemini AI
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
* `cli/bsi_cli.py`: The `click`-based CLI. This is the primary user-facing tool for managing the service, providing commands to `init-db`, `start-grpc-server`, `process-local-statements`, `view-statements`, and `reprocess-statement`.
* `core/orchestrator.py`: The heart of the processing logic. It manages the processing of a single file (`process_single_file`) by selecting the correct converter. It also contains the logic for scanning the local input directory (`process_local_directory`).
* `core/services.py`: The business logic layer that sits between the interfaces (CLI/gRPC) and the orchestrator.
    * `process_uploaded_statement`: Handles the full lifecycle of a gRPC upload, including saving the temp file, detecting the bank, calling the orchestrator, and archiving the file.
    * `reprocess_statement_by_file_id`: Implements the complex logic for reprocessing, which involves finding the original file in the archive based on DB data, re-parsing it, and updating the database records.
    * `_detect_bank`: A helper to sniff file headers (CSV or XLSX) to determine the bank format.
* `core/converters/`: A directory of bank-specific parsers.
    * `deutsche_bank_converter.py`: Parses the specific CSV format from Deutsche Bank.
    * `bank_of_america_converter.py`: Parses a specific XLSX format from Bank of America.
* `grpc_api/`: Defines and implements the external gRPC interface.
    * `protos/bsi.proto`: The Protobuf definition file, defining `ProcessBankStatement` and `ExtractDeutscheBankAccounts` services.
    * `service.py`: The Python implementation of the gRPC server that listens for and handles incoming requests.
* **Data Layer (Shared DB):**
    * `core/database/`: Contains `database.py` (engine/session management) and `models.py` (SQLAlchemy ORM definitions).
    * `core/io_handlers/`: Modules for all database interaction. `sql_writer.py` handles saving all parsed data, and `sql_reader.py` is used by the `view-statements` CLI.
* **Enrichment Helpers (DB-Driven):**
    * `core/branch_helper.py`: A singleton class that queries the `bank_accounts` table to find the correct `branch_name` based on an account number.
    * `core/invoice_parser_helper.py`: A singleton class that queries `bank_accounts` -> `business_units` -> `invoice_rules` to fetch the correct regex patterns (prefix/length) for a given account, then uses them to extract invoice numbers from raw transaction text.
* `scripts/map_management.py`: A set of utility scripts, exposed via the CLI, for development and analysis (e.g., analyzing static mapping files, extracting unique DB accounts from archives).
* `config/settings.yaml`: The central configuration file.

---

## 3. How a Bank Statement is Processed

The BSI processes files via two main workflows: gRPC upload (real-time) or local directory scanning (batch).

### Workflow 1: gRPC Upload

This is the primary flow for new, individual files.

1.  **Ingestion:** The `grpc_api/service.py` server receives a `ProcessBankStatementRequest`, which contains the `filename` and raw `file_content` (bytes).
2.  **Temp Storage:** `core/services.process_uploaded_statement` saves the file bytes to a temporary location specified by `paths.temp_files_dir`.
3.  **Bank Detection:** `services._detect_bank` is called. It opens the temp file and sniffs its content (e.g., checking for `'Account No'` and `'Bank/Branch'` in a CSV header for Deutsche Bank) to identify the bank format.
4.  **Orchestration:** `core/orchestrator.process_single_file` is called with the temp file path and the detected `bank_name`.
5.  **Conversion & Enrichment:** The orchestrator selects the correct converter (e.g., `deutsche_bank_converter.parse_statement`) from its `CONVERTER_MAP`. The converter:
    a.  Reads the file (e.g., `pandas.read_csv`).
    b.  Groups transactions by `Account No`.
    c.  For each transaction, it calls `InvoiceParserHelper.extract_invoices` to find invoice numbers by querying the live `invoice_rules` table in the DB.
    d.  For each account, it calls `BankBranchHelper.get_branch_name` to find the branch name from the `bank_accounts` table.
    e.  It structures this data into a `summary` (header) and `transactions` (lines) for each account.
6.  **Database Write:** The structured data is passed to `core/io_handlers/sql_writer.save_statement_data`. This function:
    a.  Creates one `BsiStatementFile` record for each account's `summary`.
    b.  Inserts all `BsiTransaction` records, linking them via the new `file_id`.
    c.  Upserts the `opening_balance` and `closing_balance` into the `daily_account_balances` table, using a unique constraint on (account, currency, date).
7.  **Archiving:** If the process is successful, `services.process_uploaded_statement` copies the temp file to a permanent, date-stamped archive location (e.g., `/ai/assets/csv/bsi/deutsche_bank/2025/11/11.04.2025.csv`).
8.  **Cleanup:** The temporary file is deleted.

### Workflow 2: Local Directory Scan

This flow is triggered by the `process-local-statements` CLI command.

1.  **Trigger:** `cli/bsi_cli.py` calls `core/orchestrator.process_local_directory`.
2.  **Scan:** The orchestrator walks the nested directory structure defined in `paths.local_input_dir` (e.g., `.../input/<bank_name>/<year>/<month>/<file.csv>`).
3.  **Process:** For each file found, it calls `orchestrator.process_single_file` using the directory name as the `bank_name`.
4.  **Conversion & Storage:** Steps 5 and 6 from the gRPC workflow are executed. (Note: This workflow does not include the Archiving step, as the files are already in a structured location).

### Workflow 3: Reprocessing

This flow is triggered by the `reprocess-statement --file-id <UUID>` CLI command.

1.  **Trigger:** `cli/bsi_cli.py` calls `core/services.reprocess_statement_by_file_id` with the UUID.
2.  **Find Record:** The service queries the `bsi_statement_files` table for the provided `file_id` to get its `source_bank_name` and `statement_date`.
3.  **Find Archive File:** It constructs the expected archive file path (e.g., `/ai/assets/csv/bsi/deutsche_bank/2025/11/11.04.2025.csv`) based on the bank and date.
4.  **Re-Parse:** It calls the correct converter (e.g., `deutsche_bank_converter.parse_statement`) to parse the *entire* archived file. This file may contain data for *multiple* accounts.
5.  **Find Related Records:** The service queries the DB again to find *all* `BsiStatementFile` records that share the same `source_bank_name` and `statement_date`. It builds a map of `{account_number: existing_file_id}`.
6.  **Update Database:** It iterates through the newly parsed data, account by account:
    a.  It finds the `existing_file_id` for the current account using the map.
    b.  It calls `sql_writer.save_statement_data` in **reprocess mode** (passing the `existing_file_id`).
    c.  The `sql_writer` first **deletes all existing `BsiTransaction` records** for that `file_id` before inserting the new ones and updating the `BsiStatementFile` header. This ensures no duplicate data.

---

## 4. Key Features & Functionality

* **gRPC API**: Provides a `ProcessBankStatement` endpoint for real-time, programmatic file uploads and a `ExtractDeutscheBankAccounts` helper endpoint.
* **Modular Bank Converters**: A clean `core/converters` architecture allows new banks to be supported by adding a new parser file.
* **CLI Management Suite**: A robust `click` CLI for all administrative tasks, including database initialization, manual processing, data viewing, and reprocessing.
* **Live DB-Driven Enrichment**:
    * **Invoice Parsing**: Uses `InvoiceParserHelper` to query the shared `invoice_rules` table (managed by CAA) to dynamically extract invoice numbers from transaction text.
    * **Branch Resolution**: Uses `BranchHelper` to query the shared `bank_accounts` table to resolve cryptic branch codes into human-readable branch names.
* **Robust Reprocessing**: A dedicated service (`reprocess-statement`) allows developers to update parsing logic, run the reprocessing command, and fix data in the database without manual intervention.
* **Daily Balance Tracking**: Populates and maintains a `daily_account_balances` table, providing a clean, time-series-ready view of account balances.
* **File Archiving**: Automatically archives successfully processed uploads to a permanent, date-structured directory for auditing and reprocessing.

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

* **`grpc_server`**: Port (`50052`) and max worker settings.
* **`database`**: The DSN connection string for the PostgreSQL database.
* **`paths`**:
    * `local_input_dir`: Path for the `process-local-statements` command.
    * `temp_files_dir`: Location for transient upload files.
    * `generated_stubs_path`: Shared path for gRPC stub files.
* **`logging`**: Log level, directory, and filenames.

---

## 7. Command-Line Interface (CLI) Usage

The main entry point is `bank_statement_ingestor.py`.

[bash]
# Initialize the database (drops and recreates BSI tables)
./bank_statement_ingestor.py init-db

# Start the gRPC server to listen for uploads
./bank_statement_ingestor.py start-grpc-server

# Process all files in the local input directory
./bank_statement_ingestor.py process-local-statements

# View ingested statements with filters
./bank_statement_ingestor.py view-statements
./bank_statement_ingestor.py view-statements --account "00039429000"
./bank_statement_ingestor.py view-statements --filename "11.04.2025"

# Reprocess a specific statement file from the archive
./bank_statement_ingestor.py reprocess-statement --file-id "a1b2c3d4-..."

# --- Utility Commands ---

# (Re)Generate the static invoice rule JSON from the company CSV
./bank_statement_ingestor.py generate-invoice-map

# Analyze local statement files against the static rule JSON
./bank_statement_ingestor.py analyze-account-map deutsche_bank

# Extract unique account details from local files
./bank_statement_ingestor.py extract-db-accounts
[/bash]

## 8. Future Enhancements

* **Activate Bank of America Converter**: The `bank_of_america_converter.py` is implemented, but it is currently commented out in the `core/orchestrator.CONVERTER_MAP`. The immediate next step is to test this converter with real files, ensure the `_detect_bank` service in `core/services.py` reliably identifies its specific XLSX format, and activate it in the map to support BoA files from gRPC uploads.

* **Database-Driven Bank Detection**: The current `_detect_bank` service in `core/services.py` uses hardcoded `if/elif` logic to identify bank formats based on unique column names. This is brittle and requires a code deployment to add new banks.
    * **Solution**: This logic should be migrated to the database. A new table (e.g., `bsi_bank_profiles`) could store detection rules, such as `{"file_type": "csv", "required_columns": ["Account No", "Bank/Branch"]}` and map them to a specific converter (e.g., "deutsche_bank"). The `_detect_bank` function would then query this table and loop through the profiles, making the system extensible without new code.

* **Tighter CAA Feedback Loop**: Currently, BSI is a "write-only" service; CAA consumes its data but cannot send feedback. This limits BSI's ability to learn or self-correct.
    * **Solution**: A new gRPC endpoint should be added to BSI, such as `ReparseTransactionInvoices(transaction_id)`. When the CAA `RuleEngine` applies a payment but finds that BSI failed to extract the invoice (i.e., `bsi_transaction.extracted_invoice_numbers` is empty), CAA could call this endpoint. The BSI service would then re-fetch that single transaction's `raw_bank_text` and re-run the `InvoiceParserHelper` on it, updating the `extracted_invoice_numbers` field. This is far more efficient than the current file-level `reprocess-statement` command.

* **Deprecate Static Map Utilities**: The CLI includes commands like `generate-invoice-map` and `analyze-account-map` which rely on static files (`company_details.csv`, `account_to_invoice_rules.json`). However, the live runtime logic (in `core/invoice_parser_helper.py`) correctly queries the `invoice_rules` table from the shared database.
    * **Solution**: These static utilities are misleading and are development relics. They should be removed to eliminate the "dual source of truth." A new utility, e.g., `analyze-db-rules`, could be created to perform a gap analysis by comparing local statement files against the *live database rules*, which would provide actual, actionable insights.