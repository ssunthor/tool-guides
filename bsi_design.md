# Bank Statement Ingestor (BSI) - Architecture & Design Document

**Doc_Type:** Architecture & Design
**Category:** (I) Ingestor
**Name:** Bank Statement Ingestor
**Acronym:** BSI
**Version:** 0.2.9
**Date:** 2026-10-26
**Author:** Martin Kecka & Gemini AI
**Status:** Active

---

## 1. Introduction & Purpose 📜

The **Bank Statement Ingestor (BSI)** is a foundational ingestion microservice within the SERA/GIA enterprise AI ecosystem. It is the authoritative single source of truth for all incoming bank cash transactions and daily bank account ledger balances across European, North American, Latin American, and Asia-Pacific banking operations.

Operating as an automated, multi-protocol ingestion and enrichment pipeline, BSI addresses a core operational challenge in financial automation: global financial institutions transmit statement files in archaic, proprietary, and structurally disparate formats—ranging from SWIFT MT940 messages and ANSI BAI2 cash management files to custom Excel XLSX workbooks and online banking portal CSV exports. BSI continuously discovers, ingests, sniffs, normalizes, enriches, and persists these statements into a standardized PostgreSQL relational schema.

The data processed and persisted by BSI serves as the primary prerequisite for the downstream **Cash Application Assistant (CAA)**. CAA consumes BSI transactions to automate matching incoming customer payments against open invoices and remittance advices, orchestrating the creation and application of standard receipts in Oracle Financials ERP.

Its primary functional pillars include:
- **Multi-Protocol Statement Ingestion:** Continuous background polling of remote SFTP dropboxes with dynamic rush-hour intervals, high-throughput gRPC upload endpoints for programmatic ingestion, and developer CLI tooling for local batch processing.
- **Deterministic Bank Format Sniffing:** Automated detection of bank origin and format based on byte-level content signatures, structural markers, and dynamic database account routing (`_detect_bank_from_file`).
- **Multi-Format Dialect Parsing:** Native parsers for SWIFT MT940 (Deutsche Bank, Danske Bank, Itaú Unibanco, UBB Bank, JSC OTP Bank), ANSI BAI2 (Bank of America US, EMEA, and APAC), Excel XLSX workbooks, and structured CSV exports.
- **Intelligent Lockbox & Addenda Explosion:** Reconciles Bank of America BAI2 summary transactions against raw `.dat` lockbox, ACH, and wire addenda files (`BofaEnricher`), exploding batch summaries into granular check line items with check numbers, payer names, and remittance metadata while strictly isolating regional lockboxes (e.g., Canadian Lockbox `919150`).
- **Scope-Aware Invoice Extraction Engine:** Multi-tier regular expression engine (`InvoiceParserHelper`) that queries live database rules scoped to specific Business Units to extract invoice and credit memo numbers from unstructured narrative text with OCR error repair.
- **Central Validation & Noise Pruning:** Connects via gRPC to the Cash Management Connector (CMC) to validate candidate invoice numbers against the central financial database (`invoice_validator.py`), eliminating spurious noise (telephone numbers, postal codes, routing references) and reverse-enriching missing payer names from client account metadata.
- **Idempotency & Smart Reprocessing:** Deduplication mechanisms to reject duplicate uploads, combined with an in-place reprocessing engine (`services.reprocess_statement_by_file_id`) that updates parsing logic while strictly preserving stable transaction UUIDs and pruning orphaned records.

---

## 2. Architecture & Ecosystem Integration 🏗️

BSI is engineered as a headless, modular Python service composed of an automated ingestion manager, an orchestration pipeline, format-specific converters and parsers, database I/O handlers, external connectors, and a gRPC server exposing both domain-specific upload operations and SERA standard telemetry endpoints.

### 2.1. Core Architectural Principles & Touchpoints

- **Single Source of Truth:** BSI acts as the authoritative financial ledger entrypoint for bank cash movements across corporate accounts. Downstream assistants (CAA) and reporting tools depend exclusively on BSI tables (`bsi_statement_files`, `bsi_transactions`, `daily_account_balances`).
- **Deterministic & Auditable Ingestion:** Operates with zero non-deterministic or probabilistic transformations at the ingestion boundary. All format sniffing, dialect parsing, regular expression matching, and mathematical balance reconciliations follow strict deterministic logic.
- **Non-Destructive Smart Reprocessing:** Changes to parser rules or business unit invoice patterns can be applied retroactively to archived vault files without breaking existing relational linkages. Transaction UUIDs are preserved across re-runs.
- **Remote Bank SFTP (`external_services.oracle_sftp`):** SFTP server hosting bank statement drops and Bank of America `.dat` addenda files. Polled by `IngestionManager` using `SFTPConnector` (`10.150.22.55:22`).
- **Cash Management Connector (CMC - `connectors.cmc`):** gRPC service running on port `50053`. BSI invokes `ValidateInvoices` to verify candidate invoice numbers and `GetInvoicesByBan` for Billing Account Number (BAN) customer metadata lookups.
- **Shared PostgreSQL Database (`caa_assistant_db`):** Shared persistence layer. BSI owns the write lifecycle for `bsi_statement_files`, `bsi_transactions`, `daily_account_balances`, and `bsi_ingestion_audit`. It queries master financial structures including `physical_bank_accounts`, `bu_bank_assignments`, `cmd_business_units`, and `invoice_rules`.
- **Cash Application Assistant (CAA):** Downstream consumer. CAA's `bsi_processing_service` continuously monitors `bsi_transactions` where `status = 'unprocessed'`, advancing them through the automated payment application state machine.
- **Application Management Utility (AMU):** SERA ecosystem supervisor. Interacts with BSI's gRPC server (`sera_standard_service.py`) on port `50052` to probe health matrix dependencies, systemd unit status (`bsi-grpc.service`, `bsi-poller.service`), and version telemetry.
- **GIA Gateway / External Clients:** Transmits programmatic bank statement uploads via the `BankStatementIngestorService.UploadStatement` gRPC RPC on port `50052`.

### 2.2. Ecosystem Integration & Data Flow

```mermaid
graph TD
    subgraph Sources ["Inbound Statement Sources"]
        SFTP["Remote Bank SFTP<br/>(Oracle SFTP 10.150.22.55:22)"]
        GRPC_IN["gRPC Client / GIA Gateway<br/>(UploadStatement RPC :50052)"]
        LOCAL_IN["Local Batch CLI<br/>(/ai/bank_statement_ingestor/input)"]
    end

    subgraph BSI_Core ["Bank Statement Ingestor (BSI)"]
        POLLER["IngestionManager<br/>(Rush Hour Dynamic Poller)"]
        GRPC_SRV["BSI gRPC Server<br/>(Port 50052)"]
        CLI["BSI Click CLI<br/>(bank_statement_ingestor.py)"]
        
        DETECT["Bank Sniffer & Router<br/>(_detect_bank_from_file)"]
        ORCH["Core Orchestrator<br/>(PARSER_REGISTRY)"]
        
        subgraph Parsers ["Converters & Parsers"]
            CONV_DB["Deutsche MT940 / CSV"]
            CONV_DANSKE["Danske MT940"]
            CONV_BOFA["BofA BAI2 (US / EMEA / APAC)"]
            CONV_XLSX["BofA US/EMEA XLSX"]
            CONV_INTL["Itaú / UBB / JSC OTP MT940"]
        end
        
        subgraph Enrich ["Enrichment & Validation Engine"]
            BOFA_ENRICH["BofaEnricher<br/>(Lockbox & Addenda Explosion)"]
            INV_HELPER["InvoiceParserHelper<br/>(Account-Scoped Regex)"]
            BRANCH_HELPER["BankBranchHelper<br/>(DB / Static BIC Lookup)"]
            INV_VALID["InvoiceValidator<br/>(Batch CMC Validation & Pruning)"]
        end
        
        WRITER["SQL Writer<br/>(Smart Merge & Deduplication)"]
        VAULT["Permanent Storage Vault<br/>(/ai/assets/vault/bank_statement_ingestor)"]
    end

    subgraph External_Ecosystem ["SERA Ecosystem Integrations"]
        CMC["Cash Management Connector (CMC)<br/>(gRPC Port 50053)"]
        DB[("Shared PostgreSQL<br/>(caa_assistant_db :5432)")]
        CAA["Cash Application Assistant (CAA)<br/>(bsi_processing_service)"]
        AMU["Application Management Utility (AMU)<br/>(SeraAppService Telemetry)"]
    end

    SFTP -->|Scheduled Poll| POLLER
    GRPC_IN -->|Stream Bytes| GRPC_SRV
    LOCAL_IN -->|Manual Scan| CLI

    POLLER --> DETECT
    GRPC_SRV --> DETECT
    CLI --> DETECT

    DETECT --> ORCH
    ORCH --> Parsers
    
    Parsers --> BOFA_ENRICH
    Parsers --> BRANCH_HELPER
    Parsers --> INV_HELPER

    BOFA_ENRICH -.->|Addenda Files| SFTP
    BOFA_ENRICH -.->|BAN Lookup| CMC
    INV_VALID -->|Validate Invoices| CMC

    Parsers --> WRITER
    WRITER -->|Insert / Update| DB
    WRITER -->|Trigger Validation| INV_VALID
    INV_VALID -->|Update Validated / Pruned| DB
    
    ORCH -->|Archive Source + Addenda| VAULT
    
    DB -.->|Poll Unprocessed TXs| CAA
    GRPC_SRV -.->|GetDeepHealth / GetAppInfo| AMU
```

---

## 3. Module File Map 📂

An exhaustive structural breakdown of the `/ai/bank_statement_ingestor` repository scope:

```plaintext
/ai/bank_statement_ingestor
├── bank_statement_ingestor.py                      # Master CLI entrypoint; loads config, sets up logging, delegates to Click group
├── bootstrap_helper.py                             # Runtime environment bootstrap; auto-activates project virtualenv and re-executes interpreter
├── cli/                                            # Command-line interface definitions and command sub-groups
│   ├── bsi_cli.py                                  # Main Click group; initializes database engine and attaches core, dev, analysis, and module commands
│   └── commands/                                   # Granular Click command implementations
│       ├── analysis_cmds.py                        # QA inspection commands: validate-rules, qa-extraction, payer-qa, code-qa
│       ├── db_cmds.py                              # Database maintenance commands: init-db, inspect-tx, view-statements, ingestion-stats, cleanup-duplicates
│       ├── dev_cmds.py                             # Developer diagnostic utilities: test-sftp-parse, local-parse
│       ├── ingestion_cmds.py                       # Core runtime commands: upload-statement, reprocess-statement, start-poller
│       └── module_cmds.py                          # SERA standard module observability commands: module info, module check-health
├── config/                                         # Central application configuration
│   └── settings.yaml                               # Ecosystem v1-universal configuration (DSNs, endpoints, bank profiles, poller settings)
├── core/                                           # Core business logic, engine services, and statement processing pipeline
│   ├── analysis/                                   # Diagnostic, quality assurance, and rule analysis tool implementations
│   │   ├── ingestion_stats.py                      # Analyzes recent file ingestion events, timelines, and addenda file linkages
│   │   ├── invoice_qa.py                           # Simulates invoice extraction coverage, candidate pruning, and noise reduction against CMC
│   │   ├── local_parser.py                         # Parses local statement files and exports structured JSON for development inspection
│   │   ├── payer_qa.py                             # Scans statements to identify transactions where payer name / counterparty extraction failed
│   │   ├── rule_validator.py                       # Cross-references bank accounts in statement files against DB PhysicalBankAccount and InvoiceRule setup
│   │   └── transaction_code_analyzer.py            # Aggregates transaction codes across local files and dumps raw samples for unmapped codes
│   ├── branch_helper.py                            # BankBranchHelper singleton; resolves bank branch names from DB and static routing maps
│   ├── connectors/                                 # Outbound client communication interfaces
│   │   ├── __init__.py
│   │   ├── cmd_connector.py                        # gRPC client for CMC (ValidateInvoices, GetInvoicesByBan, GetOpenInvoices)
│   │   └── sftp_connector.py                       # Paramiko-based client for listing, downloading, and connecting to remote bank SFTP
│   ├── constants.py                                # Transaction code dictionaries (BAI2 codes, MT940 maps, Deutsche and Danske constants)
│   ├── converters/                                 # Bank statement structure converters (orchestrator parse_statement interface)
│   │   ├── bank_of_america_apac_converter.py      # Bank of America APAC BAI2 converter (Sydney Branch AUD/EUR/USD)
│   │   ├── bank_of_america_emea_converter.py      # Bank of America EMEA BAI2 converter
│   │   ├── bank_of_america_us_converter.py        # Bank of America US BAI2 converter
│   │   ├── bank_of_america_us_xlsx_converter.py   # Bank of America US/EMEA custom Excel XLSX converter with run date extraction
│   │   ├── danske_bank_converter.py               # Danske Bank MT940 converter with multi-sheet aggregation
│   │   ├── deutsche_bank_converter.py             # Deutsche Bank AG MT940 converter with :86: subtag decomposition
│   │   ├── deutsche_csv_converter.py              # Deutsche Bank AG portal CSV converter with chronological balance linking
│   │   ├── itau_bank_converter.py                 # Itaú Unibanco S.A. MT940 converter with checking + investment balance summing
│   │   ├── jsc_otp_bank_converter.py              # JSC OTP Bank MT940 converter with slash-tag parsing
│   │   └── ubb_bank_converter.py                  # United Bulgarian Bank (UBB) MT940 converter with slash-delimited field mapping
│   ├── database/                                   # Persistence layer abstractions and ORM definitions
│   │   ├── database.py                             # SQLAlchemy engine creation, scoped session generator (get_db_session), and DSN loader
│   │   └── models.py                               # SQLAlchemy ORM models (bsi_statement_files, bsi_transactions, daily_account_balances, bsi_ingestion_audit, master tables)
│   ├── enrichers/                                  # Domain-specific statement enrichment engines
│   │   └── bofa_enricher.py                        # BofaEnricher; reconciles BAI2 summary transactions against lockbox, ACH, and wire .dat addenda files
│   ├── ingestion_manager.py                        # IngestionManager; automated SFTP polling engine with rush-hour intervals and delta deduplication
│   ├── invoice_parser_helper.py                    # InvoiceParserHelper thread-safe singleton; multi-tier regex engine with account-scoped rule caching
│   ├── invoice_validator.py                        # Batch validation engine against CMC; prunes invalid candidate noise and reverse-enriches missing payers
│   ├── io_handlers/                                # Database reading and writing abstractions
│   │   ├── sql_reader.py                           # Queries statements, transactions, and global ingested filename sets (get_existing_filenames_global)
│   │   └── sql_writer.py                           # Statement persistence, smart merge, orphan deletion, and bsi_ingestion_audit logging
│   ├── orchestrator.py                             # Central pipeline orchestrator; maintains PARSER_REGISTRY and routes file execution
│   ├── parsers/                                    # Low-level syntax parsers extracting raw statement fragments
│   │   ├── bofa_apac_parser.py                     # Specialized APAC BAI2 dialect parser inheriting BofA International/EMEA tag extraction
│   │   ├── bofa_bai2_parser.py                     # Base BAI2 state machine parser (records 01, 02, 03, 16, 49, 88, 98, 99)
│   │   ├── bofa_emea_parser.py                     # Specialized EMEA BAI2 dialect parser extracting /REF-, UK Sort Codes, and FX RATE-terminated OBI
│   │   ├── bofa_us_parser.py                       # Specialized US BAI2 dialect parser handling Wire, ISO/SEPA, ACH, and EMEA text layouts
│   │   ├── bofa_us_xlsx_parser.py                  # Pandas-based parser for "Previous Day All Data Summary and Detail with Text Report" XLSX
│   │   ├── danske_mt940_parser.py                  # Danske Bank MT940 parser with :28C: sheet aggregation and multi-language :86: parsing
│   │   ├── deutsche_csv_parser.py                  # Deutsche Bank portal CSV parser with multi-day chunking and running balance calculation
│   │   ├── deutsche_mt940_parser.py                # Deutsche Bank MT940 parser with Tag :86: (?00-?33) decomposition and SEPA tag extraction
│   │   ├── itau_mt940_parser.py                    # Itaú Unibanco MT940 parser handling Brazilian decimal formatting and investment blocks
│   │   ├── jsc_otp_mt940_parser.py                 # JSC OTP Bank MT940 parser with Russian ruble slash-tag structure
│   │   └── ubb_mt940_parser.py                     # UBB Bank MT940 parser with slash-delimited :86: mapping
│   └── services.py                                 # Core business services: manual upload lifecycle, vault reprocessing, and safe invoice re-extraction
├── docs/                                           # Documentation and release history
│   ├── bsi_design.md -> /ai/assets/guides/tool_guides/bsi_design.md # Symlink to central master design document
│   └── CHANGELOG.yaml                              # Structured YAML release history parsed by AMU and SeraStandardService
├── grpc_api/                                       # gRPC interface layer and server entrypoints
│   ├── bsi_service.py                              # Implementation of BankStatementIngestorServiceServicer (UploadStatement)
│   ├── generate_stubs.py                           # Dynamic Protobuf compiler generating Python stubs on server startup
│   ├── protos/                                     # Protocol buffer definition files
│   │   └── bsi.proto                               # Service definition for UploadStatement RPC
│   ├── sera_standard_service.py                    # SERA ecosystem standard gRPC service (SeraAppService) for AMU health and telemetry
│   └── services.py                                 # Multi-servicer gRPC aggregator binding BsiService and SeraStandardService to port 50052
├── scripts/                                        # Dumping ground for one-time operational, repair, and reconciliation scripts (ephemeral, not part of runtime)
├── tests/                                          # Test and QA directories
│   ├── input/                                      # Test input statement files grouped by bank
│   ├── output/                                     # Output JSON dumps from local_parse dev runs
│   └── qa/                                         # Output directory containing timestamped QA extraction reports
├── test_parser.py                                  # Standalone local test runner for statement parsers
└── utils/                                          # Common system utilities
    ├── config_loader.py                            # YAML configuration loader with environment overrides
    └── logging_setup.py                            # Timed rotating file handlers and console logging setup
```

---

## 4. Core Processing Lifecycle ⚙️

BSI supports four distinct runtime execution lifecycles: **Automated SFTP Polling**, **Manual / gRPC Statement Upload**, **Smart Reprocessing**, and **Database-Only Rule Re-application**.

### 4.1. Boot & Initialization Sequence

1. **Environment Bootstrap (`bootstrap_helper.bootstrap_and_reexecute_if_needed`):** Executing `bank_statement_ingestor.py` verifies execution within the designated Python virtual environment, dynamically re-executing under the project venv interpreter if invoked from the host environment.
2. **Configuration & Logging (`utils/config_loader.py` & `logging_setup.py`):** `load_config()` parses `config/settings.yaml`. `setup_logging()` initializes a root logger with a `StreamHandler` (`INFO`) and two `TimedRotatingFileHandler` instances rotating at midnight (`bsi_app.log` at `DEBUG`, `bsi_err.log` at `WARNING`, `backupCount=7`). Noisy third-party loggers (`paramiko`, `urllib3`, `httpx`) are clamped to `WARNING`.
3. **Database Engine Init (`core/database/database.init_database_engine`):** Invoking any Click command triggers `@cli.group()`, calling `database.init_database_engine(config)` to bind the global SQLAlchemy `_engine` and `_SessionFactory` to `database.write_dsn` with connection pre-ping enabled.
4. **gRPC Stub Compilation (`grpc_api/generate_stubs.check_and_generate_stubs`):** When `start-grpc-server` is invoked, BSI compares the modification timestamps (`os.path.getmtime`) of `bsi.proto` and `sera_standard.proto` against the compiled `_pb2.py` files in `/ai/assets/protos/bank_statement_ingestor/generated`. If missing or outdated, `grpc_tools.protoc` compiles them and injects `__init__.py` files up the directory tree before starting the `grpc.server` thread pool.

### 4.2. Workflow A: Automated SFTP Polling (`IngestionManager.run_ingestion_cycle`)

```mermaid
stateDiagram-v2
    [*] --> IngestionTrigger

    state IngestionTrigger {
        SFTP_Poll: Automated SFTP Poller (IngestionManager)
        GRPC_Upload: Programmatic gRPC Upload (UploadStatement)
        Local_Scan: Local Batch CLI Scan (process_local_directory)
        Reprocess_Cmd: Reprocess Statement CLI (reprocess_statement)
    }

    IngestionTrigger --> ContentInspection: File Delivered to Temp Staging

    state ContentInspection {
        SniffBytes: First 8KB Sniffing (_detect_bank_from_file)
        DetectFormat: MT940 / BAI2 / CSV / XLSX
        ResolveProfile: Map to Parser Registry (orchestrator.PARSER_REGISTRY)
    }

    ContentInspection --> AddendaSync: If Bank of America (_sync_bofa_addenda)
    AddendaSync --> Conversion: Download Matching .dat Files
    ContentInspection --> Conversion: Other Banks

    state Conversion {
        ParseStructure: Extract Accounts, Dates, Balances
        DecomposeTX: Extract Lines, References, Tags
        ExplodeLockbox: Explode Check Batches (BofaEnricher)
        PreEnrich: Extract Invoices (InvoiceParserHelper) & Lookup Branch (BankBranchHelper)
    }

    Conversion --> StorageArchive: Archive to Permanent Vault
    
    state StorageArchive {
        ComputeFilename: Generate Canonical Filename
        WriteVault: Date-Partitioned Vault Directory
    }

    StorageArchive --> DatabasePersist: Call sql_writer.save_statement_data

    state DatabasePersist {
        CheckDuplicate: Verify Account + Date + Currency
        UpsertHeader: bsi_statement_files
        MergeTransactions: bsi_transactions (Preserve UUIDs)
        UpsertBalance: daily_account_balances
        AuditLog: bsi_ingestion_audit
    }

    DatabasePersist --> CMCValidation: Trigger invoice_validator.validate_and_prune

    state CMCValidation {
        BatchCandidates: Group Extracted Invoices
        InvokeCMC: gRPC ValidateInvoices (Batch 50)
        PruneNoise: Discard Invalid Matches
        BackfillPayer: Infer Missing Payer from Client/Group
    }

    CMCValidation --> [*]: Ingestion Completed (Ready for CAA)
```

1. **Wake & Window Calculation:** `start-poller` instantiates `IngestionManager` and enters an infinite loop. On each cycle, `run_ingestion_cycle()` calculates `cutoff_date = today - timedelta(days=lookback_days)` (default: `7` days).
2. **Global Deduplication Pre-Check:** Queries `sql_reader.get_existing_filenames_global()`, which returns the union of all `source_filename` values in both `bsi_statement_files` and `bsi_ingestion_audit`. **Circuit Breaker:** If the database query fails, the cycle immediately aborts to prevent duplicate ingestion.
3. **SFTP Discovery:** Opens a single `SFTPConnector` session using RSA private key authentication. Iterates over all `bank_profiles` where `active: true`. For each profile:
   - Scans `remote_path` (and automatically appends `/archive` and `/archive2` subdirectories if `"archive"` is not already in `remote_path`).
   - Filters remote filenames against `profile.filename_pattern` regex, skips files already in `existing_files`, and extracts embedded dates (`^(\d{8})`, `_(\d{8})`, or `_(\d{6})`) to discard files older than `cutoff_date`.
4. **BofA Addenda Pre-Sync:** If new statement files are queued and the profile defines `addenda_path` (`bofa/production`), BSI scans `addenda_path` with an extended lookback (`cutoff_date - 5 days`) and downloads any missing `.dat` files matching `_(\d{8})_` (`%m%d%Y`) into `temp_files_dir`.
5. **Download & Empty File Gate:** Downloads each new statement file to `temp_files_dir`. If `os.path.getsize(local_path) == 0`, logs `FAILED_EMPTY` to `bsi_ingestion_audit`, adds the filename to `existing_files`, deletes the temp file, and continues.
6. **Orchestration, Validation & Vaulting:**
   - Calls `orchestrator.process_single_file()` -> parses via the bank's converter -> writes to PostgreSQL via `sql_writer.save_statement_data()`.
   - Logs `PROCESSED` (or `FAILED_PARSE` / `FAILED_CRITICAL`) to `bsi_ingestion_audit`.
   - Calls `invoice_validator.validate_and_prune(config, file_ids)` to verify candidates against CMC.
   - Copies the raw statement file (and any `.dat` files in `temp_files_dir` for `bank_of_america_us`) into the permanent date-partitioned vault: `{vault_dir}/{bank_key}/{YYYY}/{MM}/{filename}`.
   - Removes the temporary statement file from `temp_files_dir`.
7. **Adaptive Sleep:** Checks current UTC time against `rush_hour_start` (`06:30`) and `rush_hour_end` (`12:30`). Sleeps for `rush_interval_seconds` (`300s`) during rush hours or `interval_seconds` (`1800s`) during standard hours.

### 4.3. Workflow B: Manual / gRPC Statement Upload (`services.process_manual_upload`)

1. **Inbound Reception:** Via CLI (`upload-statement --file`) or gRPC (`UploadStatement`), the file is written to `temp_files_dir`.
2. **Format Sniffing (`_detect_bank_from_file`):** Determines the exact profile key using a multi-stage inspection pipeline (detailed in Section 5.1). Maps specialized parser keys (`bank_of_america_us_xlsx`, `deutsche_bank_csv`, etc.) to their canonical bank name (`CANONICAL_BANK_NAMES`) for vaulting and DB storage.
3. **Conditional Addenda Sync:** If the detected profile is `bank_of_america_us` or `bank_of_america_us_xlsx`, connects to SFTP and syncs the last 7 days of `.dat` addenda files into the file's directory (`_sync_bofa_addenda`).
4. **Parse & Dynamic Filename Standardization:** Executes `converter.parse_statement(local_file_path)` to extract account data and the earliest statement date (or `file_run_date_utc` for BofA XLSX). To prevent portal export filename collisions (e.g., `BSDB (18).txt`) from overwriting files in the vault:
   - **Deutsche Bank (`deutsche_bank`, `deutsche_bank_csv`):** Adds `+1 day` to the statement date (aligning with portal export conventions) and renames the file to `DB_MT940_{YYYYMMDD}_{HHMMSS}{ext}` or `DB_CSV_{YYYYMMDD}_{HHMMSS}{ext}`.
   - **Bank of America (`bank_of_america_*`):** Renames to `{bofa_us|bofa_emea|bofa_apac}_{first_account}_{YYYYMMDD}_{HHMMSS}{ext}`.
   - **Other Banks:** Preserves `original_filename`.
5. **Vault Archiving:** Copies the renamed file and any `.dat` addenda into `{vault_dir}/{canonical_name}/{YYYY}/{MM}/`.
6. **Database Write & Duplicate Protection:** Calls `sql_writer.save_statement_data(..., force_update=override)`. For gRPC uploads, `override` is strictly `False`. If all account/date combinations in the file already exist in `bsi_statement_files`, `save_statement_data` returns `[]`, and `process_manual_upload` rejects the upload with `"DUPLICATE_FILE: All accounts in this statement have already been processed for this date."`
7. **Validation & Audit:** Triggers `invoice_validator.validate_and_prune(config, file_ids)` and records `"PROCESSED"` (`"Manual Upload"`) in `bsi_ingestion_audit`.

### 4.4. Workflow C: Smart Reprocessing (`services.reprocess_statement_by_file_id`)

1. **Vault Lookup:** Triggered via CLI (`reprocess-statement --file-id <UUID>` or `--filename <name>`). Queries `BsiStatementFile` to retrieve `source_bank_name`, `source_filename`, `statement_date`, and `receiving_account_number`, locating the archived file at `{vault_dir}/{bank_name}/{YYYY}/{MM}/{filename}`.
2. **Format Auto-Switching:** Inspects the vaulted file extension and header. If `bank_name` is `bank_of_america_us` or `bank_of_america_emea` and the file ends in `.xlsx`/`.xls`, switches the processor to `bank_of_america_*_xlsx`. If `bank_name` is `deutsche_bank` and the first line contains `"Account Currency"`, switches to `deutsche_bank_csv`.
3. **Account-Scoped Re-parse:** Re-parses the vaulted file (for BofA XLSX, `bank_of_america_us_xlsx_converter` automatically locates `.dat` addenda in the same `{vault_dir}/bank_of_america_us/{YYYY}/{MM}` directory) and isolates the dictionary matching `record.receiving_account_number`.
4. **Smart Merge (`sql_writer.save_statement_data`):**
   - Loads existing `BsiTransaction` rows for `existing_file_id` into `existing_tx_map` keyed by `(float(amount), str(transaction_date), debit_credit_indicator)`.
   - Updates `BsiStatementFile` header balances, credits/debits totals, and addenda metadata in place.
   - For each newly parsed transaction, pops the first matching ORM object from `existing_tx_map[match_key]` if present, updates all mutable fields (`payer_name`, `extracted_invoice_numbers`, `charges_amount`, etc.) in place while **preserving the primary key `transaction_id`** (preventing broken foreign keys in CAA's `cash_application_process`), and resets `invoice_validation_status = 'UNCHECKED'`.
   - Inserts unmatched new transactions and deletes any remaining orphaned ORM transactions in `existing_tx_map` that were not matched.
   - Upserts `DailyAccountBalance` via PostgreSQL `ON CONFLICT (account_number, currency, statement_date) DO UPDATE`.
5. **Re-validation:** Runs `invoice_validator.validate_and_prune(config, [file_id])`.

### 4.5. Workflow D: Database-Only Rule Re-application (`services.reapply_invoice_parsing_rules`)

1. **Trigger Condition:** Executed during maintenance cycles or rule updates via operational scripts to re-extract invoice numbers across previously ingested transactions without re-reading or re-parsing vaulted raw statement files.
2. **Cache Clearance:** Instantiates `InvoiceParserHelper` and invokes `invoice_helper.clear_cache()` to flush cached account-to-BU regular expressions.
3. **Target Query:** Queries `BsiTransaction` joined with `BsiStatementFile` (optionally filtered by `source_bank_name`).
4. **Narrative Text Reconstruction:** If `tx.raw_bank_text` is empty, BSI dynamically reconstructs the extraction source from `tx.bank_description`, `tx.remittance_purpose`, and MT940 tags `?20` through `?29`, updating and backfilling `tx.raw_bank_text` in place.
5. **Rule Re-extraction & Fallback:** Evaluates `invoice_helper.extract_invoices(extraction_source, account_number)` alongside the fallback pattern `r'(?:IN|INV|CR|CM)[.:\s-]*\d{6,}'`. Merges and deduplicates newly extracted candidates.
6. **In-Place DB Update:** If the newly discovered candidate list differs from `tx.extracted_invoice_numbers`, updates the array in PostgreSQL and commits.

---

## 5. Detailed Business Logic, Rules & HITL 🧠

BSI is a fully automated, headless ingestor; **interactive Human-in-the-Loop (HITL) workflows are not applicable for this module** (all HITL ambiguity resolution occurs downstream in CAA). However, BSI implements deterministic detection, enrichment, aggregation, and extraction rules.

### 5.1. Bank & Format Auto-Detection Rules (`_detect_bank_from_file`)

When a file is uploaded manually or via gRPC, BSI determines its `parser_id` using the following priority order:
1. **Binary / Text Header Inspection (Non-Excel files, first 8,192 bytes, UTF-8 with UTF-16 null-byte fallback):**
   - **Deutsche Bank CSV (`deutsche_bank_csv`):** Contains `"Account Currency"`, `"GVC"`, and `"HVP - "`.
   - **Bank of America BAI2 (`bank_of_america_apac` / `bank_of_america_emea` / `bank_of_america_us`):** Starts with `01,` and contains `,2,`. Routes to `bank_of_america_apac` if `"apac"` is in the lowercase filename, `bank_of_america_emea` if `"emea"` is in the lowercase filename, otherwise `bank_of_america_us`.
   - **MT940 Variants:** Identified by `{1:F`, `\n:20:`, `\r:20:`, or starting with `:20:`. Routed by BIC substring: `"DEUT"` -> `deutsche_bank`; `"DABA"` or `"DANSKE"` -> `danske_bank`; `"ITAU"` -> `itau_unibanco`; `"UBBS"` -> `ubb_bank`; `"OTPV"` -> `jsc_otp_bank`.
2. **Configured Filename Regex Match:** Tests `os.path.basename(file_path)` against each profile's `filename_pattern` in `settings.yaml`.
3. **Excel Content & Database Institution Routing (`.xlsx`, `.xls`):**
   - Reads the first 25 rows via Pandas. Requires `"Account Number"` and `"BAI Code"` in the header row (and `BofaUsXlsxParser` strictly validates that `"previous day all data summary and detail"` appears in the first 5 rows).
   - Extracts the first row's `Account Number`, strips non-digits via `clean_account_number()`, and queries `PhysicalBankAccount` joined with `FinancialInstitution`. If `institution.ingestion_key` contains `'emea'`, routes to `bank_of_america_emea_xlsx`; otherwise defaults to `bank_of_america_us_xlsx`.
4. **Filename Substring Fallback:** Checks for `"DB_MT940"`, `"ITAUMT940"`, `"UBBMT940"`, `"JSCMT940"`, or `"MT940_" + "U"`.

### 5.2. Bank-Specific Parsing & Daily Statement Aggregation Rules

#### Summary Parser Matrix

| Bank / Institution | Converter & Parser Files | Native Format | Balance Integrity Tolerance |
| :--- | :--- | :--- | :--- |
| **Deutsche Bank AG** | `deutsche_bank_converter.py`<br/>`deutsche_mt940_parser.py` | SWIFT MT940 | `abs(open + credits - debits - close) < 0.01` |
| **Deutsche Bank AG (CSV Export)** | `deutsche_csv_converter.py`<br/>`deutsche_csv_parser.py` | Bank Portal CSV | `diff < 0.05` (Final day closing validation) |
| **Danske Bank A/S** | `danske_bank_converter.py`<br/>`danske_mt940_parser.py` | SWIFT MT940 | `abs(open + credits - debits - close) < 0.01` |
| **Bank of America (US)** | `bank_of_america_us_converter.py`<br/>`bofa_us_parser.py` / `bofa_us_xlsx_parser.py` | ANSI BAI2 / XLSX | Record 03 / Record 49 Reconciled (`diff == 0`) |
| **Bank of America (EMEA)** | `bank_of_america_emea_converter.py`<br/>`bofa_emea_parser.py` | ANSI BAI2 | Record 03 / Record 49 Reconciled (`diff == 0`) |
| **Bank of America (APAC)** | `bank_of_america_apac_converter.py`<br/>`bofa_apac_parser.py` | ANSI BAI2 | Record 03 / Record 49 Reconciled (`diff == 0`) |
| **Itaú Unibanco S.A.** | `itau_bank_converter.py`<br/>`itau_mt940_parser.py` | SWIFT MT940 | `diff < 0.05` (Checking + Investment combined) |
| **United Bulgarian Bank (UBB)** | `ubb_bank_converter.py`<br/>`ubb_mt940_parser.py` | SWIFT MT940 | `diff < 0.05` (Per-block integrity) |
| **JSC OTP Bank** | `jsc_otp_bank_converter.py`<br/>`jsc_otp_mt940_parser.py` | SWIFT MT940 | `diff < 0.05` (Default currency RUB) |

#### Detailed Bank Implementations

##### 1. Deutsche Bank MT940 (`DeutscheMT940Parser` & `deutsche_bank_converter`)
- **Message Splitting & Grouping:** Splits messages on `\n-`. Groups fragments by `(account, currency, statement_date)`, deduplicates by `:28C:` `sequence_number_raw`, and sorts by `(statement_number, sheet_number)`. Takes `opening_balance` from the first sheet and `closing_balance` from the last sheet.
- **Integrity Check:** Mathematical check passes if `abs(open + credits - debits - close) < 0.01`.
- **Field Extraction:**
  - Splits Tag `:86:` on subtag delimiter `(\?\d{2})`. Reconstructs structured remittance purpose from `?20` through `?29`.
  - Parses SEPA tags: `EREF+` (End-to-End Ref), `SVWZ+` (Remittance), `KREF+` (Customer Ref), `DEBT+` (Debtor), `ABWA+` (Originator/Payer), `ABWE+` (Alternative Recipient), `CHRG:` (Charges).
  - Payer resolution: Prioritizes `ABWA+` over `?32/?33` for counterparty name.
  - Foreign Exchange: Extracts original currency, original amount, and exchange rate from `:61:` subfield `/OCMT/` and `:86:` markers `SETT-AMT:`, `RATE`, or `UEBER`.
  - Transaction Code `063`: Dynamically classified as `"EU Standard Payment"` (`C` / Credit) or `"Bank Fees / Charges"` (`D` / Debit).

##### 2. Deutsche Bank Portal CSV (`DeutscheCsvParser` & `deutsche_csv_converter`)
- **Statement Chunking:** Groups rows by `(acc_no, curr, iso_date)` to split multi-day portal exports into daily statements.
- **Chronological Balance Linking:** Because intermediate days in DB CSV exports contain stale `Book Balance` values, BSI chronologically links running balances: sets `opening_balance = prev_closing`, trusts calculated closing (`open + credits - debits`) on intermediate days, and validates against the CSV's reported `closing_balance` on the final day (`diff < 0.05`).
- **Field Extraction:**
  - Concatenates `Counter Party Name 1` and `Counter Party Name 2`.
  - Merges `Booking Text` with `All Transaction Details` (or `Transaction Detail Line 1..6`).
  - Extracts SEPA tags (`EREF+`, `SVWZ+`, `KREF+`, `DEBT+`, `CHRG:`) and FX fields (`Instructed Amount`, `HVP - Exchange Rate`, `SETT-AMT:`, `UEBER`).
  - Automatically calculates `calculated_charges_amount = abs(orig_amt - amt)` when original and account currencies match.

##### 3. Danske Bank MT940 (`DanskeMT940Parser` & `danske_bank_converter`)
- **Message Splitting & Aggregation:** Splits messages on `(?m)^(?=:20:)`. Aggregates paginated daily sheets by `(account, currency, statement_date)` using `:28C:` `seq/sheet` sorting, identical to Deutsche MT940 (`diff < 0.01`).
- **BIC Normalization:** Sets `sender_bic` to `DABASESX` if the account number starts with `SE`, otherwise `DABADKKK`.
- **Field Extraction:**
  - Normalizes 2-character D/C indicators (`DD`, `RD` -> `D`; `CD`, `RC` -> `C`).
  - Parses multi-language `:86:` narrative lines (`Uppdragsgivare:`, `Ordering customer:`, `Betalare:`, `Bnf.`, `Bnf.acc.`, `/Our fee`, `Mottaget belopp:`, `Beraknade avgifter:`, `DBTS`).
  - Unstructured Payer Heuristic: For transaction codes `NTRF` and `NMSC`, evaluates the first descriptive text line as a fallback payer name.

##### 4. Bank of America US BAI2 & XLSX (`BofaUsParser`, `BofaUsXlsxParser`, `bank_of_america_us_converter`)
- **BAI2 Parsing Engine:** Parses records `01` (File Header), `02` (Group Header), `03` (Account Identifier), `16` (Transaction Detail with `S`/`V` funds-type column offsets), `88` (Continuation / Summary), and `49` (Account Trailer). Divides amounts by `100` (except non-decimal currencies `JPY`, `HUF`, `KRW`).
- **Summary Identification:** Flags BAI type codes `115` and `175` as `is_summary = True` for lockbox addenda explosion.
- **Narrative Layout Classification:** Decomposes narrative text across 4 distinct banking layouts:
  1. **Wire:** `WIRE TYPE:`, `TRN:`, `ORIG:`, `ID:`, `PAYMENT DETAILS:`, `RELATED REF:`.
  2. **ISO/SEPA:** `Dbtr Nm:`, `Dbtr Addr:`, `Dbtr Bank:`, `Dbtr Client Ref:`.
  3. **ACH:** `DES:`, `INDN:`, `CO ID:`, `PMT INFO:`.
  4. **EMEA-style:** `ORG=`, `BNF=`, `OBI=`, `/REF-`.
- **Deducted Fees & FX:** Extracts `FEES: $X DEDUCTED` (setting `charges_bearer = 'BEN'` and reconstructing `original_amount = amount + fee`) and `FX: CUR AMT RATE`.

##### 5. Bank of America EMEA & APAC BAI2 (`BofaEmeaParser`, `BofaApacParser`, `bank_of_america_emea_converter`, `bank_of_america_apac_converter`)
- **Inheritance:** `BofaEmeaParser` extends the BAI2 state machine from `BofaBai2Parser`; `BofaApacParser` subclasses `BofaEmeaParser` to process international APAC feeds (`^gttcommu_APAC_PDR_\d+\.txt$`, e.g., Sydney Branch `BOFAAUSX` across `AUD`, `EUR`, and `USD`).
- **Field Extraction:**
  - Extracts `/REF-` as `bank_reference`.
  - Parses `ORG=` ordering party blocks: strips leading `1/`, extracts `3/` as address, and matches trailing `(\d{6})\s+(\d{8})$` as UK Sort Code & Account Number.
  - Extracts exchange rate and instructed amounts from `FX RATE ([A-Z]{3})/\s+([\d,.]+)\s+([\d,.]+)`.
  - Extracts `OBI=` remittance purpose via `r'OBI=(.*?)(?=\s+FX RATE|$)'` (explicitly terminating prior to inline `FX RATE` blocks to prevent exchange rate details from corrupting `remittance_purpose`), capturing `/RFB/` as `end_to_end_reference`.

##### 6. Itaú Unibanco MT940 (`ItauMT940Parser` & `itau_bank_converter`)
- **Message Splitting & Investment Aggregation:** Splits on `$` and `(?=\{1:F)`. Strips `AA+` automatic investment suffix from `:25:` account numbers. Aggregates Checking and Investment blocks for the same `(account, date, currency)` by **summing** both `opening_balance` and `closing_balance` and recalculating integrity (`diff < 0.05`).
- **Field Extraction:**
  - Converts Brazilian numeric format (`1.000,00` -> `1000.00`).
  - Classifies `:86:` narratives: extracts payer from `"TED"` transfers (`TED\s[\d\.]+\s?(.+)`), identifies payments via `"SISPAG"`, and flags `"APLIC AUT"` / `"IOF"` / `"IR"` as `"INVESTMENT"` transactions.
  - Skips invoice extraction on internal `"INVESTMENT"` rows.

##### 7. United Bulgarian Bank MT940 (`UbbMT940Parser` & `ubb_bank_converter`)
- **Message Splitting:** Splits on `$` and `(?=\{1:F)`. Validates per-block integrity (`diff < 0.05`).
- **Field Extraction:**
  - Splits `:61:` transaction reference on `//` into `customer_reference` and `bank_reference`.
  - Extracts 3-digit UBB transaction code from the start of `:86:` (`965`, `929`, `241`, `864`, `244`, etc.).
  - Maps slash-delimited segments (`/`) to remittance text, counterparty name, End-to-End reference, IBAN, and BIC. Sets `payer_name = "Internal FX"` for transaction code `NFEX`.

##### 8. JSC OTP Bank MT940 (`JscOtpMT940_Parser` & `jsc_otp_bank_converter`)
- **Message Splitting:** Splits on `$` and `(?=\{1:F)`. Default currency `RUB`. Validates per-block integrity (`diff < 0.05`).
- **Field Extraction:**
  - Parses slash-tag `:86:` structure (`/BNF/`, `/ORDP/`, `/REMI/`).
  - For `/BNF/` or `/ORDP/` strings containing double slashes `//` (e.g., `ACC... INN... // REF // ACTUAL PAYER`), extracts the segment after the final `//` as `counterparty_name`.

### 5.3. Bank of America Addenda Reconciliation (`BofaEnricher`)

For `bank_of_america_us` and `bank_of_america_us_xlsx`, `BofaEnricher` reconciles statement transactions against auxiliary `arlockbox*.dat` files (`CHECK`, `ACH`, `WIRE`):

```plaintext
BAI2 Statement: Credit $12,450.00 (Summary Deposit, Ref: 919150)
                      │
                      ▼
Addenda Sync: arlockboximportgenbai2_919150_20261012_103000.dat
                      │
                      ▼
[Batch Match: Batch Total Cents == Statement Amount Cents ($12,450.00)]
                      │
     ┌────────────────┼────────────────┐
     ▼                ▼                ▼
Check 1: $4,200.00   Check 2: $6,100.00 Check 3: $2,150.00
Payer: Acme Corp     Payer: GlobalTech  Payer: Beta LLC
Inv: INV-100234      Inv: INV-500211    Inv: INV-892100
```

1. **Addenda File Parsing (`_parse_addenda_file`):** Parses record types `5` (Batch Header), `6` (Item Detail: extracts amount in cents, 9-digit routing number, remitter account, and check/wire reference), `4` (Remittance Overflow: tokenizes strings into `INVOICE`, `BAN`, `NOISE`, or `UNIDENTIFIED`), and `7`/`8`/`9` (Batch/File totals in cents).
2. **Token Normalization (`_normalize_token`):**
   - Canadian Lockbox format `^9(\d{7,9})CA$` -> `('INVOICE', 'INV' + digits)`
   - Billing Account Number (BAN) `(?<![A-Z])T(\d{6})(?!\d)` -> `('BAN', 'T' + digits)`
   - Billing Account Number (BAN) `[09]?(?:C)(\d{6,9})` (excluding `CM`/`CR`) -> `('BAN', 'C' + digits)`
   - Invoice `(?:[09]?(?:INV|1NV|UNV|IN|V)|[09]IN)(?:O|0)?(\d{7,13})` -> `('INVOICE', 'INV' + digits)`
   - Credit Memo `[09]?(?:CM|CR)(\d{7,12})` -> `('INVOICE', 'CM' + digits)`
   - Pure numeric `^\d{7,13}$` (stripping leading `9` or `0` if length >= 9) -> `('INVOICE', 'INV' + digits)`
3. **CMC BAN Prefetching (`_prefetch_ban_metadata`):** Collects all unique BANs across loaded `.dat` files and calls `cmd_connector.get_invoices_by_ban(ban)` once, caching `{client, invoices}` in `BofaEnricher._meta_cache['ban_map']`.
4. **Summary Lockbox Explosion (`is_summary == True`, BAI Codes `115`, `175`):**
   - **Strict Lockbox ID Guard:** Extracts the lockbox ID from the `.dat` filename (`arlockboximportgenbai2_(\d+)_`) and compares it against the transaction's `bank_reference`. If they do not match (e.g., US vs. Canadian lockbox `919150`), the file is skipped to prevent cross-contamination of identical deposit amounts.
   - Matches the statement deposit amount (in cents) against either `file_total_cents` or an individual `batch_total_cents`.
   - **Explosion:** Replaces the single summary transaction (`is_summary=True`) with $N$ individual check transactions (`is_summary=False`), populating each check's exact `amount`, `bank_reference` (check number), `payer_name` (from BAN lookup), `payer_sort_code` (routing), `payer_account_number`, `extracted_invoice_numbers`, and `addenda_metadata`.
   - If a BAN was provided on the check without explicit invoice numbers, `BofaEnricher` automatically matches any open invoice under that BAN whose `balance` or `original_amt` matches the check amount within `$0.02`.
   - **Graceful Canadian Lockbox Logging:** If lockbox `919150` has no matching `.dat` file yet, logs at `INFO` rather than `WARNING`.
5. **1:1 ACH / Wire Enrichment (`is_summary == False`):** Matches non-summary credits (excluding codes `354` and `399`) where the `.dat` item's `raw_amount_string` starts with `str(int(round(tx['amount'] * 100)))`, enriching `extracted_invoice_numbers`, `payer_name`, `payer_sort_code`, and `payer_account_number` in place.

### 5.4. Scope-Aware Invoice Extraction Engine (`InvoiceParserHelper`)

`InvoiceParserHelper` is a thread-safe singleton that extracts structured document numbers from unstructured transaction narrative text:

```plaintext
Raw Bank Text: "TRANSFER FROM ACME CORP INV: 89012345 12.10.2026 REF: 9912"
                      │
                      ▼
Account Resolution: physical_bank_accounts -> cmd_links -> cmd_bu_name ("GTT_US")
                      │
                      ▼
Rule Fetch: invoice_rules WHERE cmd_bu_name = 'GTT_US' AND is_active = TRUE
Rule Config: [Prefix: "INV", Length: 11] -> Expected Regex: INV + 8 digits
                      │
                      ▼
Pattern Execution Pipeline:
1. Strict Regex:           INV[.:\s-]*\d{8}           --> Match: INV89012345
2. Split Digit Repair:     INV[.:\s-]*\d{4}\s+\d{4}   --> (Handles space inserts)
3. Split Prefix Repair:    I\s*N\s*V[.:\s-]*\d{8}     --> (Handles "I N V")
4. Date Suffix Repair:     INV\d{8}(?=\d{1,2}\.\d)    --> (Detaches glued dates)
5. Truncated Prefix:       NV[.:\s-]*\d{8}            --> (Repairs dropped 'I')
                      │
                      ▼
Raw Candidate Output: ["INV89012345"]
```

1. **Strict Account-to-BU Scoping (`_query_rules`):**
   - Cleans the receiving account number (`re.sub(r'\D', '', acc_no)`).
   - Queries `PhysicalBankAccount` (where `is_active == True`) and resolves linked Business Units via `cmd_links` (falling back to `oracle_links` if no CMD link exists).
   - Queries active `InvoiceRule` rows (`rule_type` in `'invoice'`, `'credit_memo'`) for those BUs. If a rule has `prefix == ""` but defines a numeric `example` matching `length`, infers the first 4 digits (`ex_str[:4]`) as the prefix. Caches rules per account in `_rules_cache`.
2. **Multi-Pass Regex Extraction (`_compile_and_find`):** For each `(prefix, length, bu_name)` rule (where `num_chars_strict = length - len(prefix)`), runs 8 extraction methods:
   - **`Strict` (`trusted=True`):** `rf'{prefix_pat}([.:\s-]*)(\d{{{num_chars_strict}}})(?!\d)'`
   - **`SplitDigit` (`trusted=True`):** Repairs whitespace-split numbers where `len(p1 + p2) == num_chars_strict` (`rf'{prefix}(?:[.:\s-]*)([0-9]+)\s+([0-9]+)(?![0-9])'`).
   - **`SplitPrefix` (`trusted=True`):** Repairs whitespace inside multi-character prefixes (e.g., `I N V 123456`).
   - **`DateSuffixRepair` (`trusted=True`):** Extracts invoices immediately concatenated with a date (`(?=\d{1,2}\.\d)`).
   - **`Contained` (`trusted=True`):** Extracts `prefix + digits` even when followed by trailing digits.
   - **`PrefixNorm` (`trusted=True`):** Cross-matches `IN` <-> `INV` variations while normalizing the output to the BU's canonical prefix.
   - **`TruncatedPrefix` (`trusted=True`):** Matches prefixes missing their first character (e.g., `NV123456` for `INV123456`) when bounded by whitespace or punctuation.
   - **`PrefixInjection` (`trusted=False`):** Matches bare numeric sequences of length `length - len(alpha_part)` (when `>= 4` digits), verifying any numeric prefix constraint and checking `_check_preceding_conflict` to ensure the digits aren't preceded by another known alpha prefix, then injects `alpha_part.upper()`.
3. **Substring Deduplication (`_deduplicate_substrings`):** Sorts candidates by `(len(value), trusted)` descending and drops any candidate that is a substring of an already-kept longer candidate.
4. **Unconfigured Account Fallback:** In `deutsche_bank_converter`, `deutsche_csv_converter`, and `danske_bank_converter`, if `invoice_helper.has_configured_rules(acc_no)` is `False`, BSI executes a generic fallback regex: `r'(?:IN|INV|CR|CM)[.:\s-]*\d{6,}'`.

### 5.5. Post-Ingestion Validation, Pruning & Payer Backfilling (`invoice_validator.py`)

Immediately after `sql_writer.save_statement_data()` commits new or reprocessed files, `validate_and_prune(config, file_ids)` executes:
1. Queries all `BsiTransaction` rows for `file_ids` where `invoice_validation_status == 'UNCHECKED'` or `status == 'unprocessed'`.
2. Collects all unique strings across `tx.extracted_invoice_numbers` and validates them against CMC (`cmd_connector.validate_invoices`) in batches of `50`.
3. **Noise Pruning:** Any candidate not returned by CMC is removed from `tx.extracted_invoice_numbers`.
4. **Payer Reverse-Enrichment (`_resolve_payer_name`):** If `tx.payer_name` is empty/null and the transaction has validated invoices:
   - If all validated invoices belong to **1 unique `client`**, sets `tx.payer_name = client`.
   - Else if all validated invoices belong to **1 unique `client_group`**, sets `tx.payer_name = client_group`.
5. Marks `tx.invoice_validation_status = 'VALIDATED'` and commits.

### 5.6. Bank Branch Name Resolution (`BankBranchHelper`)

To ensure standard reporting and audit clarity across downstream modules:
1. Queries `physical_bank_accounts.branch_name` in PostgreSQL using the normalized account number.
2. If unmapped in the database, falls back to `STATIC_BIC_MAP`, resolving known international SWIFT BICs and US routing numbers:
   - `051000017`: Bank of America N.A., Henrico Branch
   - `111000012`: Bank of America N.A., Dallas Branch
   - `BOFAUS3N`: Bank of America N.A., New York Branch
   - `BOFACATT`: Bank of America Canada, Toronto Branch
   - `BOFAGB22`: Bank of America N.A., London Branch
   - `BOFAIE3X`: Bank of America Europe DAC, Dublin Branch
   - `BOFAIN4X`: Bank of America N.A., India Branch
   - `BOFANLNX`: Bank of America Europe DAC, Amsterdam Branch
   - `BOFAAUSX`: Bank of America N.A., Sydney Branch
   - `DABADKKK`: Danske Bank A/S
   - `DABASESX`: Danske Bank A/S, Sweden Branch
3. Caches resolved branch names in `BankBranchHelper._cache` (`threading.Lock()`) for zero-latency retrieval during statement conversion.

### 5.7. Human-in-the-Loop (HITL) Integration

*Not applicable for this module.*

BSI is a fully automated, headless backend ingestion service. It does not provide an interactive HITL operator workbench. All exception resolution workflows, ambiguous payment reviews, and manual rule overrides occur downstream within the **Cash Application Assistant (CAA)** user interface. Operational interventions on BSI (such as forcing a re-upload or purging duplicate records) are performed via CLI by system administrators.

---

## 6. AI, LLM & Machine Learning Execution 🤖

*Not applicable for this module.*

BSI executes strictly deterministic financial parsing algorithms, compiled Protobuf contracts, high-throughput regular expressions, and mathematical integrity checks. AI/LLM operations (such as remittance advice interpretation using Gemma-4-31B-it via the SERA AI Factory) are intentionally isolated downstream within the Cash Application Assistant (CAA) to maintain strict auditability, deterministic execution, and low latency at the ingestion boundary.

---

## 7. Data Models, State, Storage & Auditing 🗄️

### 7.1. PostgreSQL Schema (`caa_assistant_db`)

BSI persists statement files, parsed transaction lines, daily ledger balances, and operational audit records in `caa_assistant_db` across 4 dedicated ORM tables, while reading configuration and linking attributes from 2 shared master tables:

#### Tables Owned & Written by BSI

| Table Name | ORM Class | Primary Key & Indexes/Constraints | Key Columns & Types | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`bsi_statement_files`** | `BsiStatementFile` | • PK: `file_id` (`UUID`)<br/>• Indexes: `source_filename`, `source_bank_name`, `sender_bic`, `statement_date`, `account_currency`, `receiving_account_number`<br/>• Cascade: `delete-orphan` to `bsi_transactions` | `source_filename` (`String(255)`), `source_bank_name` (`String(100)`), `file_format` (`String(20)`), `sender_bic` (`String(20)`), `statement_date` (`Date`), `statement_number` (`Integer`), `our_company_name` (`String(255)`), `account_currency` (`String(10)`), `receiving_account_number` (`String(50)`), `total_credit_amount` (`Numeric(15,2)`), `total_debit_amount` (`Numeric(15,2)`), `branch_name` (`String(100)`), `opening_balance` (`Numeric(15,2)`), `closing_balance` (`Numeric(15,2)`), `addenda_source_files` (`JSONB`), `ingestion_timestamp_utc` (`DateTime(timezone=True)`) | Stores one header record per account per statement date. Tracks balances, sender BICs, and linked `.dat` addenda files. |
| **`bsi_transactions`** | `BsiTransaction` | • PK: `transaction_id` (`UUID`)<br/>• FK: `file_id` -> `bsi_statement_files.file_id`<br/>• Indexes: `transaction_date`, `transaction_code`, `payment_instrument_id`, `payer_iban`, `payer_sort_code`, `payer_account_number`, `bank_reference`, `end_to_end_reference`, `customer_reference_kref`, `status`, `invoice_validation_status` | `transaction_date` (`Date`), `entry_date` (`String(10)`), `amount` (`Numeric(15,2)`), `debit_credit_indicator` (`String(2)`), `account_currency` (`String(10)`), `original_amount` (`Numeric(15,2)`), `original_currency` (`String(10)`), `fx_rate` (`Numeric(18,10)`), `transaction_code` (`String(50)`), `transaction_code_description` (`String(255)`), `bank_operation_code` (`String(50)`), `charges_amount` (`Numeric(15,2)`), `charges_currency` (`String(10)`), `charges_bearer` (`String(10)`), `calculated_charges_amount` (`Numeric(15,2)`), `payer_name` (`String(255)`), `payer_iban` (`String(50)`), `payer_bic` (`String(20)`), `payer_sort_code` (`String(20)`), `payer_account_number` (`String(50)`), `payer_address` (`String(255)`), `bank_reference` (`String(100)`), `bank_reference_2` (`String(100)`), `customer_reference` (`String(100)`), `end_to_end_reference` (`String(100)`), `customer_reference_kref` (`String(100)`), `ultimate_debtor_debt` (`String(100)`), `bank_description` (`Text`), `remittance_purpose` (`Text`), `raw_bank_text` (`Text`), `extracted_invoice_numbers` (`ARRAY(String)`), `status` (`String(50)`), `invoice_validation_status` (`String(50)`), `mt940_tags_raw` (`JSONB`), `addenda_metadata` (`JSONB`), `is_summary` (`Boolean`) | Stores individual standardized transaction lines, raw bank tags/addenda JSONB, and validated invoice arrays. |
| **`daily_account_balances`** | `DailyAccountBalance` | • PK: `balance_id` (`UUID`)<br/>• FK: `bsi_file_id` -> `bsi_statement_files.file_id`<br/>• Indexes: `account_number`, `currency`, `statement_date`<br/>• UniqueConstraint: `uq_daily_balance` on `('account_number', 'currency', 'statement_date')` | `account_number` (`String(50)`), `currency` (`String(10)`), `statement_date` (`Date`), `opening_balance` (`Numeric(15,2)`), `closing_balance` (`Numeric(15,2)`), `recorded_at_utc` (`DateTime(timezone=True)`) | Deduplicated daily ledger balance table upserted on every statement save (`ON CONFLICT DO UPDATE`). |
| **`bsi_ingestion_audit`** | `BsiIngestionAudit` | • PK: `audit_id` (`UUID`)<br/>• Indexes: `source_filename`, `source_bank_name`, `status` | `source_filename` (`String(255)`), `source_bank_name` (`String(100)`), `file_size_bytes` (`BigInteger`), `status` (`String(50)`), `error_message` (`Text`), `ingestion_timestamp_utc` (`DateTime(timezone=True)`) | Immutable operational execution audit log preventing repeated polling of failed, empty, or duplicate files. |

#### Shared Master Tables Read by BSI
- **`financial_institutions`:** Maps `institution_id` to `ingestion_key` (used to distinguish US vs. EMEA BofA XLSX files).
- **`physical_bank_accounts`:** Maps `bsi_account_number` and `currency` (`uq_physical_acct_curr`) to `branch_name`, `institution_id`, `cmd_links`, and `oracle_links`.
- **`cmd_account_links` & `oracle_account_links`:** Joins `physical_bank_accounts` to `cmd_business_units` (`cmd_bu_name`) and `oracle_business_units` (`oracle_bu_name`).
- **`invoice_rules`:** Stores active regex configuration (`cmd_bu_name`, `rule_type`, `prefix`, `length`, `example`, `is_active`).

### 7.2. State Enumerations

- **`BsiTransaction.status`:** Initialized to `'unprocessed'` by BSI; transitioned by CAA (`'processed'`, `'ignored'`, `'applied_...'`, etc.).
- **`BsiTransaction.invoice_validation_status`:** `'UNCHECKED'` (on initial insert or smart reprocess update) -> `'VALIDATED'` (after `invoice_validator.validate_and_prune` completes).
- **`BsiIngestionAudit.status`:** `'PROCESSED'`, `'FAILED_EMPTY'`, `'FAILED_DETECT'`, `'FAILED_PARSE'`, `'FAILED_CRITICAL'`, `'SKIPPED_DUPLICATE'`.

### 7.3. In-Memory Caching & Thread Safety

- **`InvoiceParserHelper._rules_cache`:** Thread-safe dictionary guarded by `threading.Lock()` (`_cache_lock`), mapping `cleaned_account_no` -> `{'invoice': List[CachedRule], 'credit_memo': List[CachedRule]}`. Cleared via `clear_cache()` during CLI inspection/reprocessing.
- **`BankBranchHelper._cache`:** Thread-safe dictionary guarded by `threading.Lock()` (`_cache_lock`), mapping `account_key` -> `branch_name`.
- **`BofaEnricher._meta_cache`:** Class-level dictionary caching `{ban: {'client': ..., 'invoices': [...]}}` across `.dat` addenda enrichment runs.

### 7.4. File System Storage Layout

- **Permanent Vault (`/ai/assets/vault/bank_statement_ingestor`):** Partitioned as `{vault_dir}/{canonical_bank_name}/{YYYY}/{MM}/{filename}` alongside any supporting `.dat` addenda files.
- **Transient Staging (`/ai/assets/tmp/bank_statement_ingestor`):** Stores active SFTP downloads and gRPC upload payloads during processing; cleaned up in `finally` blocks.
- **Local Batch Input (`/ai/bank_statement_ingestor/input`):** Used by `orchestrator.process_local_directory`.
- **QA & Dev Simulation (`/ai/bank_statement_ingestor/tests`):** Contains `input/<bank_name>`, `output/<bank_name>` (JSON dumps), and `qa/<bank_name>/qa_report_YYYYMMDD_HHMMSS.txt`.

---

## 8. APIs, Observability & Communication Contracts 📡

### 8.1. Inbound Domain gRPC API (`bsi.proto` on `:50052`)

**Package:** `bank_statement_ingestor`  
**Service:** `BankStatementIngestorService`

| RPC Method | Request Message | Response Message | Operational Description |
| :--- | :--- | :--- | :--- |
| **`UploadStatement`** | `UploadStatementRequest`<br/>• `string filename = 1`<br/>• `bytes file_content = 2` | `UploadStatementResponse`<br/>• `bool success = 1`<br/>• `string message = 2`<br/>• `int32 transactions_processed = 3` | Writes `file_content` to `temp_files_dir/filename`, invokes `services.process_manual_upload(config, local_path, override=False)`, and removes the temp file. Rejects duplicate account/date uploads. |

### 8.2. Universal SERA Standard Contract (`sera.standard.v1.SeraAppService` on `:50052`)

BSI implements the universal SERA standard contract (`sera.standard.v1.SeraAppService` in `grpc_api/sera_standard_service.py`), centrally maintained and deployed by the Application Management Utility (`amu_design.md`). It exposes standardized endpoints (`GetAppInfo`, `GetDeepHealth`, `GetFeedback`, `GetChangelog`, `GetDocs`) enabling AMU and the SAM UI to self-discover BSI's metadata, deep dependency health matrix (`postgresql`, `cmc`, `oracle_sftp`), changelog history, and design documentation.

### 8.3. Outbound Client Contracts (`core/connectors/cmd_connector.py` -> `:50053`)

Connects to `CmdConnectorServiceStub` with a `50 MB` max send/receive message length (`MAX_MESSAGE_LENGTH = 50 * 1024 * 1024`) and metadata `(('client_application', 'BSI'),)`:
- **`ValidateInvoices(ValidateInvoicesRequest(invoice_numbers=List[str]))`:** Timeout `30s`. Returns validated `invoices_data` (converted via `MessageToDict`), including `doc_no`, `client`, `client_group`, `balance`, and `original_amt`.
- **`GetInvoicesByBan(GetInvoicesByBanRequest(ban=str, months_back=6))`:** Timeout `30s`. Used by `BofaEnricher` to fetch open/closed invoices and client names for Billing Account Numbers found in `.dat` addenda.
- **`GetOpenInvoices(GetOpenInvoicesRequest(start_date=str, end_date=str))`:** Timeout `120s`. Utility method for fetching open invoices within a date window.

---

## 9. User Interface (UI) & Visualizations 🖥️

*Not applicable for this module.*

BSI is a headless backend service. User-facing views for inspecting ingested bank statements, monitoring the predictive Ingestion Schedule Widget, and triggering manual file uploads are hosted within the **Cash Application Assistant (CAA)** React SPA. Operational telemetry, health matrix probes, and systemd service statuses are rendered in the **Sera Admin Module (SAM)** web interface via AMU.

---

## 10. Background Services & Workers 🛠️

| Worker / Service Name | Entrypoint / Location | Trigger & Schedule | Concurrency & Locking | Operational Responsibility & Retry Policy |
| :--- | :--- | :--- | :--- | :--- |
| **`bsi-poller.service`** | `./bank_statement_ingestor.py start-poller`<br/>(`IngestionManager`) | Continuous `while True` loop.<br/>• **Rush Hours (`06:30`–`12:30` UTC):** `300s` (5m) sleep.<br/>• **Standard Hours:** `1800s` (30m) sleep. | Single-instance systemd service. Deduplicates globally against `bsi_statement_files` U `bsi_ingestion_audit`. | • **DB Unreachable:** Aborts cycle immediately before SFTP scan.<br/>• **Unhandled Exception:** Logs traceback and sleeps `60s` before retrying.<br/>• **Parse/Empty Failure:** Logs to `bsi_ingestion_audit` so broken files are not infinitely re-downloaded. |
| **`bsi-grpc.service`** | `./bank_statement_ingestor.py start-grpc-server`<br/>(`grpc_api.services.serve`) | Continuous gRPC server listening on `0.0.0.0:50052`. | `futures.ThreadPoolExecutor(max_workers=4)`. Singleton caches (`InvoiceParserHelper`, `BankBranchHelper`) use `threading.Lock()`. | Catches file write and processing exceptions per request, cleans up temp files, logs `FAILED_*` to `bsi_ingestion_audit`, and returns structured `UploadStatementResponse(success=False, message=...)`. |

---

## 11. CLI, Tooling & Testing 💻

### 11.1. CLI Command Reference

All CLI commands are executed via `./bank_statement_ingestor.py <command>`.

| Command | Arguments & Options | Operational Description |
| :--- | :--- | :--- |
| **`init-db`** | *None* | Drops and recreates `bsi_statement_files`, `bsi_transactions`, `daily_account_balances`, and `bsi_ingestion_audit` tables safely. |
| **`start-grpc-server`** | *None* | Compiles protobuf stubs if needed and starts the dual gRPC server on `0.0.0.0:50052`. |
| **`start-poller`** | *None* | Starts the continuous background SFTP ingestion poller daemon with dynamic rush-hour intervals. |
| **`upload-statement`** | `--file <path>` (required)<br/>`--override` (flag) | Manually detects bank format, syncs BofA addenda, archives to vault, ingests into DB, and validates invoices via CMC. |
| **`reprocess-statement`**| `--file-id <UUID>`<br/>`--filename <name>` | Re-parses archived vault file(s) and performs a smart merge preserving existing `transaction_id` UUIDs. |
| **`inspect-tx`** | `--tx-id <UUID>`<br/>`--bank-ref <snippet>` | Dumps every database column for a transaction, displays file context, clears parser cache, and runs live extraction test. |
| **`view-statements`** | `--file-id <UUID>`<br/>`--filename <str>`<br/>`--date <YYYY-MM-DD>`<br/>`--account <acc>` | Displays ingested statements, opening/closing balances, and transaction summaries matching criteria. |
| **`ingestion-stats`** | `--bank <str>`<br/>`--limit <int>` (default: 20) | Joins `bsi_statement_files` and `bsi_ingestion_audit` within a 10s window to print a color-coded ingestion timeline. |
| **`cleanup-duplicates`**| `--days <int>` (default: 14)<br/>`--account <acc>`<br/>`--delete` (flag) | Detects identical transactions matching `(account, date, amount, dc, raw_text)`, keeps the earliest, and deletes duplicates if `--delete` is passed. |
| **`analysis validate-rules`** | `--input-dir <dir>`<br/>`--test-data` (flag)<br/>`--debug` (flag) | Scans statements and cross-references accounts against `PhysicalBankAccount`, BU links, and `InvoiceRule`, flagging configuration gaps. |
| **`analysis qa-extraction`** | `--input-dir <dir>`<br/>`--test-data` (flag)<br/>`--bank <name>` | Parses files, extracts candidates with rule attribution, batch-validates against CMC, and writes reports to `tests/qa/<bank>/`. |
| **`analysis payer-qa`** | `--bank <list>`<br/>`--code <list>`<br/>`--direction <C\|D>`<br/>`--limit <int>`<br/>`--show-raw / --no-raw` | Identifies transactions where counterparty extraction failed, prioritized by Credit (`HIGH PRIORITY`) vs. Debit (`LOW PRIORITY`). |
| **`analysis code-qa`** | `--bank <list>`<br/>`--direction <CREDIT\|DEBIT\|BOTH>`<br/>`--unknown-only` (flag)<br/>`--limit <int>` | Aggregates transaction codes across local files and dumps raw `:61:`/`:86:` narrative samples for unmapped codes. |
| **`dev test-sftp-parse`** | `<bank_name>` (argument)<br/>`--list` (flag)<br/>`--file <name>`<br/>`--latest` (flag)<br/>`--all` (flag)<br/>`--days <int>`<br/>`--download-only` (flag) | Connects to remote SFTP production and archive paths, downloads matching files to `tests/input/`, and dumps parsed JSON to `tests/output/`. |
| **`dev local-parse`** | `<bank_name>` (argument) | Parses all files in `tests/input/<bank_name>`, validates candidates via CMC, simulates payer backfilling, and prints BofA enrichment stats. |
| **`module info`** | `--grpc` (flag) | Queries local `SeraAppService` gRPC endpoint to verify AMU application metadata reporting. |
| **`module check-health`** | `--grpc` (flag) | Queries local `SeraAppService` gRPC endpoint to verify deep health matrix probes against DB, SFTP, and CMC. |

### 11.2. CLI Quick Execution Examples

```bash
# ==============================================================================
# CORE OPERATIONAL COMMANDS
# ==============================================================================

# Initialize or reset BSI database tables safely (drops and recreates tables)
./bank_statement_ingestor.py init-db

# Launch the continuous background SFTP poller service
./bank_statement_ingestor.py start-poller

# Launch the BSI inbound gRPC server
./bank_statement_ingestor.py start-grpc-server

# Manually upload and ingest a bank statement (auto-detects bank and archives)
./bank_statement_ingestor.py upload-statement --file "/path/to/statement.xlsx"

# Force manual upload even if the statement date already exists
./bank_statement_ingestor.py upload-statement --file "/path/to/statement.txt" --override

# Reprocess a statement from the vault by file UUID (single account)
./bank_statement_ingestor.py reprocess-statement --file-id "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d"

# Reprocess all accounts belonging to a specific vaulted filename
./bank_statement_ingestor.py reprocess-statement --filename "DB_MT940_20261012_083000.txt"

# View ingested statements with filtering
./bank_statement_ingestor.py view-statements --account "00039429000" --date "2026-10-12"

# View recent ingestion events timeline, file sizes, and status
./bank_statement_ingestor.py ingestion-stats --limit 25

# Deep inspection of a single transaction: dumps all columns and tests live regex rules
./bank_statement_ingestor.py inspect-tx --tx-id "b1c2..."
./bank_statement_ingestor.py inspect-tx --bank-ref "INV-89102"

# Scan and clean up duplicate transactions within a lookback window
./bank_statement_ingestor.py cleanup-duplicates --days 14 --account "00039429000"
./bank_statement_ingestor.py cleanup-duplicates --days 14 --delete

# ==============================================================================
# ANALYSIS & QUALITY ASSURANCE SUITE (analysis group)
# ==============================================================================

# Validate account configuration coverage by matching DB rules to statements
./bank_statement_ingestor.py analysis validate-rules --test-data

# Simulate invoice candidate extraction, CMC validation, and pruning metrics
./bank_statement_ingestor.py analysis qa-extraction --bank "danske_bank"

# Identify credit or debit transactions where counterparty extraction failed
./bank_statement_ingestor.py analysis payer-qa --bank "deutsche_bank" --direction CREDIT --limit 10

# Scan statements for unrecognized transaction codes missing from dictionaries
./bank_statement_ingestor.py analysis code-qa --bank "itau_unibanco" --unknown-only

# ==============================================================================
# DEVELOPMENT & DIAGNOSTIC TOOLS (dev group)
# ==============================================================================

# Dry-run SFTP connection to list available files without ingesting
./bank_statement_ingestor.py dev test-sftp-parse deutsche_bank --list --days 5

# Download and parse the latest file from SFTP to local dev tests directory
./bank_statement_ingestor.py dev test-sftp-parse bank_of_america_us --latest

# Parse local statement files and dump structured JSON for inspection
./bank_statement_ingestor.py dev local-parse danske_bank

# ==============================================================================
# SERA MODULE OBSERVABILITY (module group)
# ==============================================================================

# Retrieve SERA application info and active dependencies via local gRPC
./bank_statement_ingestor.py module info

# Run deep health checks against PostgreSQL, SFTP, and bank profiles
./bank_statement_ingestor.py module check-health
```

---

## 12. Configuration, Security & Hardware Affinity 🔒

### 12.1. Configuration Reference (`config/settings.yaml`)

*(Note: Universal SERA boilerplate blocks—`module_info.*`, standard `logging.*`, and standard protobuf `paths.*`—are omitted below; see `conf_concept.md`.)*

| Configuration Key | Default / Configured Value | Purpose |
| :--- | :--- | :--- |
| `system_services.grpc` | `"bsi-grpc.service"` (`0.0.0.0:50052`, `4` workers) | Systemd unit, bind address/port, and worker count for `BankStatementIngestorService` and `SeraAppService`. |
| `system_services.poller` | `"bsi-poller.service"` | Systemd unit name for the SFTP background poller daemon. |
| `database.write_dsn` | `"postgresql://ai_assistant_user:***@localhost:5432/caa_assistant_db"` | SQLAlchemy PostgreSQL connection string. |
| `connectors.cmc` | `"localhost:50053"` | Target hostname and port for the CMD Connector (CMC) gRPC service. |
| `external_services.oracle_sftp` | `"sera-ai-on-prem@10.150.22.55:22"` (key: `"/root/keys/sftp/sera_ai_sftp_key"`) | SFTP server host, port, username, and RSA private key path for bank statements and addenda. |
| `paths.local_input_dir` | `"/ai/bank_statement_ingestor/input"` | Local batch directory scanned by `process_local_directory`. |
| `paths.temp_files_dir` | `"/ai/assets/tmp/bank_statement_ingestor"` | Transient directory for SFTP downloads and gRPC uploads. |
| `paths.vault_dir` | `"/ai/assets/vault/bank_statement_ingestor"` | Permanent date-partitioned archive for raw statements and `.dat` addenda. |
| `paths.dev_tests_dir` | `"/ai/bank_statement_ingestor/tests"` | Root directory for `dev` and `analysis` CLI test data and reports. |
| `poller.enabled` | `true` | Master toggle for the SFTP poller configuration block. |
| `poller.interval_seconds` | `1800` | Off-peak polling interval in seconds (30 minutes). |
| `poller.rush_interval_seconds` | `300` | Peak-hour polling interval in seconds (5 minutes). |
| `poller.rush_hour_start` / `end` | `"06:30"` / `"12:30"` | UTC start and end window (`HH:MM`) for high-frequency polling. |

#### Configured Bank Profiles (`bank_profiles`)

| Profile Key | Active | Parser ID | Remote SFTP Directory | Addenda Directory | Statement Filename Regex Pattern |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`deutsche_bank`** | `true` | `deutsche_bank` | `db_gttsftp/production/BankStatements` | *None* | `^DB_MT940_\d{14}\.txt$` |
| **`danske_bank`** | `true` | `danske_bank` | `db_gttsftp/production/BankStatements/archive2/danske` | *None* | `^MT940_\d{6}U\d{3}$` |
| **`bank_of_america_us`** | `true` | `bank_of_america_us` | `db_gttsftp/production/BankStatements/archive2/boa` | `bofa/production` | `^gtt_commu_pd_\d+\.txt$` |
| **`bank_of_america_us_xlsx`** | `true` | `bank_of_america_us_xlsx` | *None (Manual/Portal)* | *None* | `.*\.xlsx$` |
| **`bank_of_america_emea`** | `true` | `bank_of_america_emea` | `db_gttsftp/production/BankStatements/archive2/boa` | *None* | `^gttcommu_EMEA_PDR_\d+\.txt$` |
| **`bank_of_america_apac`** | `true` | `bank_of_america_apac` | `db_gttsftp/production/BankStatements/archive2/boa` | *None* | `^gttcommu_APAC_PDR_\d+\.txt$` |
| **`itau_unibanco`** | `true` | `itau_unibanco` | `db_gttsftp/production/BankStatements/archive2/accesspay/itau` | *None* | `^\d{12}ITAUMT940Multiple\.fin$` |
| **`ubb_bank`** | `true` | `ubb_bank` | `db_gttsftp/production/BankStatements/archive2/accesspay/ubb` | *None* | `^\d{12}UBBMT940Multiple\.fin$` |
| **`jsc_otp_bank`** | `true` | `jsc_otp_bank` | `db_gttsftp/production/BankStatements/archive2/accesspay/jsc` | *None* | `^\d{12}JSCMT940b?Multiple\.fin$` |

### 12.2. Security & Cryptographic Key Handling

- **SFTP Authentication:** Password authentication and SSH agent forwarding are explicitly disabled (`look_for_keys=False`, `allow_agent=False`). Authentication strictly uses the RSA private key at `/root/keys/sftp/sera_ai_sftp_key` via `paramiko.RSAKey.from_private_key_file()`.
- **Telemetry DSN Sanitization:** `SeraStandardServiceImpl.GetDeepHealth` strips database passwords from `write_dsn` and `read_dsn` using `urllib.parse` before exposing `write_sanitized_dsn` in AMU health responses.
- **Internal Network Isolation:** gRPC communication between BSI (`:50052`), CMC (`:50053`), and CAA occurs over internal localhost/private network channels.

### 12.3. Hardware Affinity & Execution Boundaries

- **Compute Profile:** CPU-bound I/O and regular expression processing service.
- **Hardware Requirements:** Requires **no GPU hardware affinity** (runs on standard on-premise application nodes alongside CAA and CMC). Memory footprint is minimal (~200MB resident memory), with high I/O efficiency.

---

## 13. Error Handling & Resilience 🛡️

1. **Duplicate Ingestion Circuit Breaker (`IngestionManager`):** Before opening an SFTP connection, `run_ingestion_cycle()` queries `get_existing_filenames_global()`. If PostgreSQL is unreachable, the cycle immediately aborts with a `CRITICAL` log rather than risking duplicate downloads and database writes.
2. **Audit-Backed Poison File Quarantine (`bsi_ingestion_audit`):** Zero-byte files (`FAILED_EMPTY`), unrecognizable manual uploads (`FAILED_DETECT`), and malformed files (`FAILED_PARSE`, `FAILED_CRITICAL`) are logged into `bsi_ingestion_audit`. Because `get_existing_filenames_global()` unions `bsi_statement_files` with `bsi_ingestion_audit`, poison files on the SFTP server are automatically skipped on subsequent polling cycles without blocking healthy files.
3. **Mathematical Balance Integrity Checks:** Every parser calculates `abs(opening_balance + total_credits - total_debits - closing_balance)` and populates `summary['integrity_check']`. Multi-sheet MT940 statements (`DanskeMT940Parser`, `DeutscheMT940Parser`), multi-block Checking+Investment statements (`ItauMT940Parser`), and multi-day CSV exports (`DeutscheCsvParser`) aggregate and link balances chronologically before evaluating variance.
4. **Non-Destructive Smart Reprocessing:** Reprocessing a statement file via `reprocess-statement` matches transactions by `(amount, transaction_date, debit_credit_indicator)` and updates existing `BsiTransaction` rows in place, preserving `transaction_id` UUIDs so downstream CAA processes and Oracle staging receipts are never orphaned.
5. **Graceful Encoding & Line-Level Fault Tolerance:** File readers use `errors='replace'` (and UTF-16 BOM/null-byte detection in `_detect_bank_from_file`) to survive non-UTF-8 characters in international remittance text. `BofaBai2Parser` wraps individual line parsing in a `try/except` block, logging the exact `[filename:line_num]` on error without crashing the entire multi-account statement file.

---

## 14. Future Enhancements & Roadmap 🚀

### 14.1. Database-Driven Bank Detection Rules
Migrate the hardcoded format-sniffing heuristics in `core/services._detect_bank_from_file` into a dedicated database table (e.g., `bsi_bank_detection_rules` linked to `financial_institutions`), enabling new bank formats and header signatures to be onboarded via configuration rather than code deployments.

### 14.2. Granular Single-Transaction Re-Extraction RPC (`ReparseTransactionInvoices`)
Expose a lightweight gRPC endpoint allowing the CAA UI or rule engine to trigger `InvoiceParserHelper` + `invoice_validator` on a single `transaction_id` (or account group) immediately after an operator updates `invoice_rules` in the CAA UI, avoiding full file reprocessing.

### 14.3. Automated Delayed Addenda Reconciliation
When a Bank of America US or Canadian (`919150`) lockbox summary deposit (`BAI 115`) is ingested before its corresponding `.dat` addenda file arrives on the SFTP server, automatically trigger `reprocess_statement_by_file_id` on the summary statement once the missing `.dat` file is detected in a subsequent polling cycle.

### 14.4. In-Memory Streaming Ingestion
Transition SFTP downloads and gRPC uploads from intermediate disk staging (`temp_files_dir`) to in-memory byte streams, eliminating ephemeral disk writes, reducing storage I/O wear, and accelerating batch throughput.

### 14.5. Direct Corporate Bank API & Webhook Connectors
Augment batch SFTP polling with real-time REST/JSON webhook connectors for modern corporate banking APIs (e.g., Bank of America CashPro API, Deutsche Bank db-API) to support real-time intraday cash visibility.

### 14.6. Distributed Tracing (OpenTelemetry W3C `trace_id`)
Inject and propagate W3C `trace_id` headers across the ingestion pipeline, linking bank statement file ingestion events directly to downstream CAA cash application audit logs and Oracle ERP receipt generation.