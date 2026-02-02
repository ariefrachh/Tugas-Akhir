```mermaid
flowchart TB
  %% =========================================================
  %% Modul G2-A: Policy-Aware Hybrid Retriever (Latest Code)
  %% =========================================================

  %% ---------- TITLE ----------
  T["🔎 <b>Modul G2-A: Policy-Aware Hybrid Retriever</b><br/><span style='font-size:12px'>Flowchart Alur Kerja — Dari Ingest Dokumen hingga Output ContextChunk</span>"]:::title

  %% ---------- PHASE 1 LABEL ----------
  P1LBL["🔧 <b>PHASE 1: INITIALIZATION</b> (One-time Setup / Batch Ingest)"]:::phase

  %% ---------- PHASE 1 ----------
  subgraph P1[" "]
    direction TB

    A1["1. 📁 Document Loading<br/><br/><b>document_loader.py</b><br/>• Scan folder <code>kbs/</code> (PDF/DOCX/XLSX)<br/>• Parse konten sesuai tipe file<br/>• Infer metadata policy + doc_type"]:::stepPink
    A1O["Output: <b>List[Document]</b><br/>(content + doc_type + metadata)"]:::outBox

    A2["2. ✂️ Chunking Strategy<br/><br/><b>embedding_generator.py</b><br/><b>SmartChunkSplitter</b><br/>• TABLE: split per rows (10 rows/chunk, header dipertahankan)<br/>• FAQ: split Q/A (fallback generic)<br/>• POLICY/SOP: split sections → split_text_by_size(1200, overlap 250)<br/>• GENERIC: TEXT_SEPARATORS + merge by size (1200, overlap 250)"]:::stepPink
    A2O["Output: <b>List[Chunk]</b><br/>(content + strict metadata)"]:::outBox

    A3["3. 🧠 Embedding Generation & Storage<br/><br/><b>embedding_generator.py</b><br/><b>EmbeddingGenerator</b><br/>• Load SentenceTransformer (all-MiniLM-L6-v2, 384-dim)<br/>• Generate embedding untuk tiap chunk<br/>• Store ke PostgreSQL (Neon)"]:::stepOrange
    A3O["Database: <code>documents</code> & <code>chunks</code><br/>chunks(content, embedding[384], metadata)"]:::dbBox

  end

  %% ---------- PHASE 2 LABEL ----------
  P2LBL["⚡ <b>PHASE 2: RETRIEVAL</b> (Runtime Query)"]:::phase

  %% ---------- PHASE 2 ----------
  subgraph P2[" "]
    direction TB

    Q0["🔎 INPUT: User Query"]:::inputBox

    B1["4. 🧠 Query Embedding<br/><br/><b>retriever.py</b><br/>• Embed query → vector(384)"]:::stepPink

    B2["5. 🧭 Vector Similarity Search (pgvector)<br/><br/>• SQL: ORDER BY embedding <=> query_vector<br/>• similarity = 1 - distance"]:::stepOrange

    B3["6. 🛡️ Policy Filter (STRICT)<br/><br/>• role ∈ allowed_roles<br/>• classification ∈ allowed_classifications<br/>• optional: system = system_filter<br/><b>STRICT:</b> jika system_filter diberikan → <u>NO fallback</u>"]:::stepPurple

    B4["7. 📉 Thresholding & Sorting<br/><br/>• SIMILARITY_THRESHOLD<br/>• MIN_CONFIDENCE_SCORE<br/>• sort by relevance_score desc"]:::stepPink

    OUT["✅ OUTPUT: <b>RetrievalResult</b><br/>List[ContextChunk]<br/><span style='font-size:12px'>metadata STRICT 4 fields: source, role, system, classification</span>"]:::outputBox
  end

  %% ---------- FLOW ----------
  T --> P1LBL --> A1 --> A1O --> A2 --> A2O --> A3 --> A3O --> P2LBL --> Q0 --> B1 --> B2 --> B3 --> B4 --> OUT

  %% ---------- STYLES ----------
  classDef title fill:#ffffff,stroke:#ffffff,color:#1f2a37,font-size:18px;
  classDef phase fill:#6d28d9,stroke:#6d28d9,color:#ffffff,font-size:13px;

  classDef stepPink fill:#ff5aa5,stroke:#ff5aa5,color:#ffffff;
  classDef stepOrange fill:#f8b26a,stroke:#f8b26a,color:#1f2a37;

  classDef stepPurple fill:#7c3aed,stroke:#7c3aed,color:#ffffff;

  classDef outBox fill:#ffffff,stroke:#93c5fd,color:#111827,stroke-dasharray: 5 3;
  classDef dbBox fill:#ffffff,stroke:#f59e0b,color:#111827,stroke-dasharray: 5 3;

  classDef inputBox fill:#e0f2fe,stroke:#38bdf8,color:#0f172a;
  classDef outputBox fill:#e0f2fe,stroke:#38bdf8,color:#0f172a;

