## RAG Confluence Retriever (CFR) - Design Document

**Version:** 0.5
**Date:** 2025-08-08
**Author:** Martin Kecka & Gemini AI
**Category:** Retriever (R)
**Acronym:** CFR (ConFluence Retriever)

---

## 1. Introduction & Purpose
The **RAG Confluence Retriever (CFR)** is a specialized `gRPC` service module within the **SERA AI** ecosystem. Its primary purpose is to act as a callable tool for the **SERA Orchestrator**, providing relevant, structured, and intelligently filtered context from the Confluence knowledge base.

When the **SERA Orchestrator** needs to answer user queries, it invokes **CFR**. **CFR** bridges the gap between a user's query and the vectorized data by employing a multi-stage process: query intent analysis, hybrid search, relevance reranking, advanced filtering, and context expansion. The goal is to deliver a highly relevant and logically structured payload that an LLM can easily synthesize into a high-quality answer.

## 2. Architecture & Integration
**Service Type:** External `gRPC` Service.
**Called by:** SERA Orchestrator (via its `ToolExecutor` component).
**Interacts with:**
- **PostgreSQL Database:** Accesses structured metadata about Confluence pages, chunks, and summaries stored by the `Confluence Ingestor (CFI)`.
- **Vector Database (ChromaDB):** Performs initial semantic similarity searches across `chunks`, `page_summaries`, and `generated_questions` (for HyDE).
- **Embedding Model Client (via `EmbeddingManager`):** Generates query embeddings using the same model as the ingestor.
- **NVIDIA NIM for Reranking (via `RerankerManager`):** Uses a specified NVIDIA NIM for reranking retrieved candidates to improve relevance.
- **Ollama LLM for Filtering (via `FilteringManager`):** Uses a general-purpose LLM (e.g., `Gemma`) to perform a fast, preliminary analysis of the user's query to determine its intent.

## 3. Core Functionality & Process Flow
The CFR service exposes a primary `gRPC` method (`GetIntelligentContext`). The process flow is as follows:

1.  **Query Intent Analysis:** Upon receiving a query, the `FilteringManager` first calls an LLM to classify the query's intent (e.g., is the user asking for a "Procedural Guide", a "Conceptual Overview", or a "Code Script"?). This classification is used later for intelligent filtering.
2.  **Query Embedding:** The input query text is converted into a vector embedding by the `EmbeddingManager`.
3.  **Candidate Retrieval (Hybrid Search):**
    - **HyDE Search:** The query embedding is used to find similar pre-generated questions in the `confluence_question_embeddings` collection. The corresponding answer chunks for these matched questions are fetched from PostgreSQL and become high-priority initial candidates.
    - **Standard Vector Search:** The query embedding is also used to search the `confluence_store` (content chunks) and `confluence_page_summaries` collections for additional relevant items.
    - **Combination:** Results are combined and deduplicated into a single candidate pool.
4.  **Re-Ranking (NVIDIA NIM):** Query-document pairs are sent to the NVIDIA NIM reranking service, which returns a relevance score for each. Candidates are then re-sorted based on these scores.
5.  **Advanced Post-Reranking Filtering:**
    - **Dynamic Score Filtering:** A dynamic threshold is calculated based on the top-scoring result. Candidates with scores significantly lower than the top score are dropped. This step only activates if the top score is above a certain confidence level, preventing overly aggressive filtering on ambiguous queries.
    - **Deterministic Intent Filtering:** The remaining candidates are filtered based on the query intent identified in Step 1. For example, if the user asked for a "Procedural Guide", chunks classified as "API Reference" might be removed, improving the precision of the final result set.
6.  **Sibling Chunk Expansion (Context Broadening):** For the top-ranked results that pass the filtering stages, the system identifies the document section (via its Table of Contents number) and fetches all other "sibling" chunks from the same section. This provides broader context around the most relevant hits.
7.  **Hierarchical Formatting & Output Generation:** The final, expanded, and sorted list is transformed into a hierarchical structure.
    - All chunks are grouped under their parent `DocumentContext`.
    - A list of all unique hyperlinks found in the retrieved content is aggregated.
    - The final structure is returned in an `IntelligentContextResponse`.

## 4. gRPC API Definition (GetIntelligentContext)

```protobuf
// Main service definition
service RAGConfluenceRetrieverService {
  // Retrieves and formats context based on a query.
  rpc GetFormattedContext(GetFormattedContextRequest) returns (GetFormattedContextResponse);

  // Retrieves structured, hierarchical context for intelligent synthesis.
  rpc GetIntelligentContext(GetIntelligentContextRequest) returns (IntelligentContextResponse);
}

// ===================================================================
// Primary RPC for Intelligent Context Retrieval
// ===================================================================

message GetIntelligentContextRequest {
  string query = 1;
  repeated string space_keys = 2;
  repeated string page_ids = 3;
}

message IntelligentContextResponse {
  enum Status {
    SUCCESS = 0;
    ERROR = 1;
    NO_RESULTS_FOUND = 2;
  }
  Status status = 1;
  repeated DocumentContext suggested_flow = 2;
  optional string aggregated_links_json = 3; // JSON string of all unique links
  optional string error_message = 4;
}

message DocumentContext {
  string source_title = 1;
  string source_url = 2;
  optional string brief_summary = 3;
  optional string detailed_summary = 4;
  repeated ChunkContext retrieved_chunks = 5;
}

message ChunkContext {
  string content = 1;
  optional string header = 2;
  optional string toc_number = 3;
}
```

Of course. My apologies for the formatting error in the previous response. Here is the continuation of the document from section 6, correctly formatted in raw markdown.

Markdown

## 5. Configuration (config/settings.yaml)
`CFR` is managed by its `config/settings.yaml`, which includes:

- `gRPC` server settings (host, port).
- PostgreSQL DSN and pool settings.
- Vector Database (`ChromaDB`) connection details.
- `embedding_model_client` configuration.
- `reranker_model_client` configuration.
- `filtering_model_client` configuration (for the query intent analysis LLM).
- Default retrieval parameters.
- Logging configuration.

## 6. Dependencies
- **Internal:** Access to data stores populated by `Confluence Ingestor`.
- **External Model Services:**
    - NVIDIA NIM for embeddings.
    - NVIDIA NIM for reranking.
    - Ollama for query filtering.
- **Python Libraries:** `grpcio`, `sqlalchemy`, `chromadb-client`, `requests`, `pyyaml`, etc.

## 7. Logging & Auditing
The service implements two distinct forms of logging, configured in the `logging` section of `settings.yaml`.

### Application Logging
- Handles general service lifecycle events, such as startup, shutdown, warnings, and errors.
- Managed by `utils/logging_setup.py`.
- Logs are written to timestamped files (e.g., `cfr_service_YYYYMMDD_HHMMSS.log`) in the `log_dir`.
- Log rotation is handled automatically based on the `log_file_keep_count` setting.

### Transaction Logging
- Provides detailed, human-readable logs for every individual `gRPC` request for auditing and debugging purposes.
- Managed by `utils/transaction_logger.py`.
- Each transaction is saved to its own file (e.g., `cfr_txn_YYYYMMDD_HHMMSS_microseconds_queryhash.log`) in the `transaction_log_dir`.
- Each log contains the full request, retrieval details (scores, types, sources), and the final formatted context.
- Rotation is managed separately based on the `transaction_log_keep_count` setting.

## 8. Future Enhancements (Considerations)

### Retrieval & Efficiency Enhancements
- **Metadata-Driven Hybrid Search:** Augment semantic search by performing a pre-filtering step on structured metadata in `PostgreSQL`. This involves filtering chunks by `document_type` (e.g., "Procedural Guide," "API Reference") or other key entities before passing the narrowed-down list of candidates to the vector database. This will improve both speed and accuracy.
- **Two-Stage Retrieval (LLM Decider):** Implement a new retrieval flow to reduce latency for simple queries. Stage 1 retrieves only page summaries. An "LLM Decider" then analyzes if the summaries are sufficient to answer the query. If not, Stage 2 performs a targeted fetch for the full-text chunks of only the necessary documents.
- **Advanced Query Transformation:** Before embedding, use an LLM to analyze and rewrite the user's query. This could involve breaking a complex question into multiple sub-queries or rephrasing conversational queries into more formal questions.
- **User Access Control (Permissions):** For a production environment, integrate with a permissions system to ensure users only see content they are authorized to access.
- **Feedback Loop for Reranker:** Introduce a mechanism for the orchestrator to provide feedback on result quality, which can be used to fine-tune the reranking model.
- **Caching Strategies:** Implement caching for frequently executed queries to reduce latency and load on the embedding and reranking models.
- **Advanced Link Handling & Chat Continuation:** Rework the link-handling mechanism to provide more clarity and reduce noise for the orchestrator.
    - **Schema Change:** Rename `aggregated_links_json` to `follow_up_links_json` to clarify that these are actionable links found *within* the content. Add a new `source_links_json` field that provides direct URLs back to the source pages and specific chunks that were returned in the context, to be used for citations.
    - **High-Priority Bug Fix (SERA Chat Continuation):** This schema change is a prerequisite to fixing a high-priority bug where SERA fails to continue a conversation after a RAG call. The full fix requires a coordinated effort:
        - **CFR:** Implement the new link output format.
        - **SERA:** The Orchestrator's `HistoryManager` must be updated to correctly parse and store these distinct link types.
        - **Development:** The fix will be developed and validated first within a dedicated **CFR testing mode** before full end-to-end integration testing with the Orchestrator.
    - **Enchance links with brief summaries from the destination page:** Instead of just returning link follow up links, add there Intellligent Link providing additonal context what is in the page.
- **Implement 'RAG Fusion' for Broad Queries:** Address the "top-heavy" retrieval problem where broad, document-level queries (e.g., "Tell me about the SERA ecosystem") tend to over-index on introductory and high-level sections, potentially missing crucial details from the middle or end of a document.
    - To mitigate this, a **RAG Fusion** or multi-query approach will be implemented. The process will involve:
        1.  **Intent Classification:** First, classify the query to determine if it's a broad, summary-style request.
        2.  **Structural Analysis:** If the query is broad, the retriever will analyze the structure of the top-ranked source document, using its Table of Contents.
        3.  **Multi-Query Generation:** An LLM will then generate several more specific, targeted sub-queries based on the document's main sections.
        4.  **Parallel Retrieval & Reranking:** All queries (the original and the generated sub-queries) will be executed in parallel against the vector store. The aggregated results will be passed through the existing reranker to select the most diverse and comprehensive set of chunks, ensuring full document coverage for the final synthesis.

### Advanced Ingestion-Driven Capabilities
These features depend on significant upgrades to the `Confluence Ingestor (CFI)` pipeline.

- **Process Confluence Attachments:** Enhance CFR to query a new `confluence_attachments_store` collection in addition to the main page store. This will allow the retriever to find and return context from documents (e.g., PDFs, DOCX files) attached to Confluence pages, significantly expanding the available knowledge base. This is fully dependent on the CFI being refactored to support attachment parsing.
- **Multi-Document Question Answering:** Evolve the ingestion process to generate complex question/answer pairs where the answer is a synthesis of information from a source page and the pages it links to. This would enable the retriever to answer questions that require multi-hop reasoning.
- **Knowledge Graph Integration:** Enhance the ingestion pipeline to extract knowledge triplets (subject, predicate, object) from text chunks. Storing this in a graph database would enable a new class of complex, multi-hop queries that traverse relationships between entities.