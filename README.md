```mermaid
flowchart TB
  %% =========================================================
  %% MODULE G2-A: Policy-Aware Hybrid Retriever (Latest Code)
  %% =========================================================

  %% ---------------------------
  %% PHASE 1: INITIALIZATION
  %% ---------------------------
  subgraph P1["PHASE 1 — INITIALIZATION (One-time Setup / Batch Ingest)"]
    direction TB

    A["📁 DocumentLoader.load_all_documents()<br/>Scan folder kbs/"] --> B["📄 Parse documents<br/>• PDF → text per page<br/>• DOCX → paragraphs<br/>• XLSX → sheets → markdown table"]
    B --> C["🧾 Infer metadata policy<br/>role, system, classification<br/>+ detect doc_type (TABLE/FAQ/POLICY/SOP/DOCUMENT)"]
    C --> D["✅ Output: List[Document]<br/>(content + doc_type + metadata)"]

    D --> E["🔁 process_and_store_all_documents(docs)"]

    E --> F["✂️ SmartChunkSplitter.split_document(doc)<br/>(doc_type-aware router)"]

    %% Chunk strategy detail
    F --> G["📌 Chunking Strategies<br/><br/>TABLE → split by rows<br/>(EXCEL_ROWS_PER_CHUNK=10, keep header)<br/><br/>FAQ → split by Q/A pattern<br/>(fallback to generic)<br/><br/>POLICY/SOP → split by headings/sections<br/>then split_text_by_size(chunk=1200, overlap=250)<br/><br/>GENERIC → TEXT_SEPARATORS + merge by size<br/>(CHUNK_SIZE=1200, OVERLAP=250)"]

    G --> H["🔒 Create STRICT metadata<br/>{source, role, system, classification}<br/>(4 fields ONLY)"]

    H --> I["🧠 EmbeddingGenerator<br/>• Load SentenceTransformer (all-MiniLM-L6-v2)<br/>• dim=384<br/>• connect PostgreSQL (Neon)"]

    I --> J["📌 Generate embeddings per chunk<br/>vector(384)"]

    J --> K["🗄️ Upsert into PostgreSQL<br/>documents table + chunks table<br/>(chunk_id, content, embedding, metadata)"]
  end

  %% ---------------------------
  %% PHASE 2: RETRIEVAL
  %% ---------------------------
  subgraph P2["PHASE 2 — RETRIEVAL (Runtime Query)"]
    direction TB

    Q["🔎 User Query"] --> QE["🧠 Embed query<br/>SentenceTransformer → vector(384)"]

    QE --> VS["📌 pgvector similarity search<br/>ORDER BY embedding <=> query_vector ASC<br/>similarity = 1 - distance"]

    VS --> PF["🛡️ Policy Filtering (STRICT)<br/><br/>WHERE metadata.role IN allowed_roles<br/>AND metadata.classification IN allowed_classifications<br/>AND (optional) metadata.system = system_filter<br/><br/>⚠️ If system_filter provided → STRICT (NO fallback)"]

    PF --> TH["📉 Thresholding + sorting<br/>SIMILARITY_THRESHOLD<br/>MIN_CONFIDENCE_SCORE<br/>sort descending by relevance_score"]

    TH --> OUT["✅ Output RetrievalResult<br/>List[ContextChunk]<br/>metadata STRICT 4 fields"]
  end
