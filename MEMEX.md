MEMEX
A living, breathing, searchable second brain that builds itself from everything you do — forever
🧭 Vision & Core Philosophy
MEMEX is named after Vannevar Bush's 1945 visionary concept of a machine that stores everything a person reads, writes, sees, and thinks — and lets them traverse it associatively, not hierarchically. This is that machine, finally built.

Unlike Obsidian, Notion, or any note-taking app — you don't feed MEMEX. MEMEX feeds itself. It passively ingests everything: your browser history, the documents you open, the articles you read, your code commits, your emails, your terminal sessions, your screenshots. It runs a local LLM over everything to build a semantic knowledge graph of your entire intellectual life. Then, when you need something — you don't search with keywords. You have a conversation with your own past self.

"What was that paper I read in March about mesh networking?" "Show me every time I was thinking about distributed systems in the last 6 months." "What ideas did I have after reading that article about Foucault?"

It's not RAG. It's not search. It is a living associative memory — completely local, completely private, and completely yours.

🏗️ System Architecture
text

┌────────────────────────────────────────────────────────────────┐
│                         MEMEX System                          │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   Ingestion Layer                       │  │
│  │                                                         │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │  │
│  │  │ Browser  │ │  File    │ │Terminal  │ │  Email   │   │  │
│  │  │ Watcher  │ │ Watcher  │ │ Watcher  │ │ Ingestor │   │  │
│  │  │(Extension│ │(inotify/ │ │(shell    │ │(IMAP     │   │  │
│  │  │ + local) │ │FSEvents) │ │ hooks)   │ │ local)   │   │  │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘   │  │
│  └───────│────────────│────────────│─────────────│─────────┘  │
│          └────────────┼────────────┘             │            │
│                       ▼                          ▼            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │               Universal Content Parser                  │  │
│  │  (PDF, HTML, Markdown, code, email, images via OCR)     │  │
│  └──────────────────────────┬──────────────────────────────┘  │
│                             │                                 │
│                             ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              Embedding Engine (local)                   │  │
│  │              (nomic-embed-text via Ollama)              │  │
│  └──────────────────────────┬──────────────────────────────┘  │
│                             │                                 │
│              ┌──────────────┼──────────────┐                  │
│              ▼              ▼              ▼                  │
│  ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐    │
│  │  Vector DB   │ │  Knowledge   │ │  SQLite Event      │    │
│  │  (ChromaDB   │ │  Graph       │ │  Store             │    │
│  │   local)     │ │  (Kuzu DB)   │ │  (Timeline)        │    │
│  └──────┬───────┘ └──────┬───────┘ └─────────┬──────────┘    │
│         └────────────────┼──────────────────  ┘               │
│                          ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │             Conversation Interface                       │  │
│  │         (Local LLM via Ollama — llama3 / mistral)       │  │
│  │         with full memory context injection              │  │
│  └──────────────────────────┬──────────────────────────────┘  │
│                             ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              UI: Terminal or Web (Svelte)               │  │
│  └─────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
🗂️ Full Directory Structure
text

memex/
│
├── memex/                           # Core Python package
│   ├── __init__.py
│   ├── main.py                      # Entry point (daemon + CLI)
│   ├── config.py                    # TOML config loader
│   │
│   ├── ingestors/
│   │   ├── __init__.py
│   │   ├── base.py                  # Abstract Ingestor class
│   │   ├── browser.py               # Reads browser history + visited pages
│   │   ├── filesystem.py            # Watches files: PDF, MD, DOCX, txt, code
│   │   ├── terminal.py              # Shell command history + terminal output
│   │   ├── email.py                 # IMAP local mailbox reader
│   │   ├── clipboard.py             # Clipboard history monitor
│   │   ├── calendar.py              # Local calendar events (iCal)
│   │   └── screenshot.py            # Periodic screenshot + OCR (optional)
│   │
│   ├── parsers/
│   │   ├── __init__.py
│   │   ├── pdf.py                   # PDF text extraction (pdfminer.six)
│   │   ├── html.py                  # Web page cleaning (readability)
│   │   ├── markdown.py              # Markdown parser
│   │   ├── code.py                  # Code file parser (tree-sitter)
│   │   ├── image.py                 # Image OCR (Tesseract)
│   │   ├── email_parser.py          # Email body + metadata
│   │   └── chunker.py               # Smart chunking strategy
│   │
│   ├── embeddings/
│   │   ├── __init__.py
│   │   ├── engine.py                # Embedding orchestrator
│   │   ├── ollama_embed.py          # nomic-embed-text via Ollama API
│   │   └── cache.py                 # Embedding cache (avoid re-embedding)
│   │
│   ├── graph/
│   │   ├── __init__.py
│   │   ├── builder.py               # Builds knowledge graph from content
│   │   ├── entity_extractor.py      # NER: people, places, concepts, urls
│   │   ├── relation_extractor.py    # LLM-based relationship extraction
│   │   ├── kuzu_client.py           # KuzuDB graph database interface
│   │   └── traverser.py             # Graph traversal + path finding
│   │
│   ├── memory/
│   │   ├── __init__.py
│   │   ├── store.py                 # ChromaDB vector store interface
│   │   ├── retriever.py             # Hybrid search (vector + graph + BM25)
│   │   ├── ranker.py                # Re-rank results by time + relevance
│   │   └── timeline.py              # Temporal memory interface
│   │
│   ├── conversation/
│   │   ├── __init__.py
│   │   ├── engine.py                # Conversation orchestrator
│   │   ├── context_builder.py       # Builds LLM context from retrieved memory
│   │   ├── ollama_chat.py           # Ollama LLM chat interface
│   │   ├── query_analyzer.py        # Parse query intent (temporal, semantic)
│   │   └── response_formatter.py   # Format with citations + sources
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── database.py              # SQLite connection
│   │   ├── migrations.py
│   │   └── queries/
│   │       ├── events.py
│   │       ├── documents.py
│   │       └── entities.py
│   │
│   └── ui/
│       ├── tui/
│       │   ├── app.py               # Textual TUI
│       │   └── screens/
│       │       ├── chat.py          # Main chat interface
│       │       ├── timeline.py      # Browse your memory timeline
│       │       ├── graph.py         # ASCII knowledge graph viewer
│       │       └── settings.py
│       └── web/
│           ├── src/
│           │   ├── routes/
│           │   │   ├── +page.svelte         # Chat interface
│           │   │   ├── timeline/+page.svelte # Timeline browser
│           │   │   └── graph/+page.svelte   # Interactive knowledge graph
│           │   └── components/
│           │       ├── ChatWindow.svelte
│           │       ├── MemoryCard.svelte    # Cited source card
│           │       ├── KnowledgeGraph.svelte # D3.js force graph
│           │       └── TimelineSlider.svelte
│           └── package.json
│
├── extension/                       # Browser extension (Chrome/Firefox)
│   ├── manifest.json
│   └── src/
│       ├── background.ts            # Capture page content + send to daemon
│       └── content.ts               # Extract article content
│
├── scripts/
│   ├── setup.sh                     # Install Ollama + pull models
│   └── migrate.py
│
├── pyproject.toml
├── Makefile
└── README.md
🗄️ Database Schema
SQL

-- Every piece of content ever ingested
CREATE TABLE documents (
    id              TEXT PRIMARY KEY,
    source_type     TEXT NOT NULL,
    -- 'browser' | 'file' | 'email' | 'terminal' | 'clipboard'
    source_path     TEXT,            -- File path or URL
    title           TEXT,
    raw_content     TEXT NOT NULL,
    clean_content   TEXT,            -- Parsed + cleaned text
    word_count      INTEGER,
    ingested_at     DATETIME NOT NULL,
    source_created_at DATETIME,      -- When the content was originally created
    language        TEXT,
    checksum        TEXT UNIQUE,     -- SHA256 — prevent duplicates
    is_embedded     BOOLEAN DEFAULT FALSE,
    is_graphed      BOOLEAN DEFAULT FALSE,
    metadata_json   TEXT             -- Source-specific metadata
);

-- Chunks of documents (for embedding)
CREATE TABLE chunks (
    id              TEXT PRIMARY KEY,
    document_id     TEXT NOT NULL REFERENCES documents(id),
    chunk_index     INTEGER NOT NULL,
    content         TEXT NOT NULL,
    token_count     INTEGER,
    chroma_id       TEXT,            -- ID in ChromaDB
    embedded_at     DATETIME
);

-- Entities extracted from documents
CREATE TABLE entities (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    entity_type     TEXT,
    -- 'person' | 'place' | 'concept' | 'technology' | 'url' | 'organization'
    first_seen      DATETIME NOT NULL,
    last_seen       DATETIME,
    mention_count   INTEGER DEFAULT 1,
    aliases         TEXT             -- JSON array of alternate names
);

-- Entity mentions within documents
CREATE TABLE entity_mentions (
    id              TEXT PRIMARY KEY,
    entity_id       TEXT NOT NULL REFERENCES entities(id),
    document_id     TEXT NOT NULL REFERENCES documents(id),
    chunk_id        TEXT REFERENCES chunks(id),
    context_snippet TEXT,            -- Surrounding text
    mentioned_at    DATETIME NOT NULL
);

-- Relations between entities (knowledge graph edges)
CREATE TABLE relations (
    id              TEXT PRIMARY KEY,
    from_entity_id  TEXT NOT NULL REFERENCES entities(id),
    to_entity_id    TEXT NOT NULL REFERENCES entities(id),
    relation_type   TEXT NOT NULL,   -- e.g. "wrote", "related_to", "cited_in"
    source_doc_id   TEXT REFERENCES documents(id),
    confidence      REAL,
    created_at      DATETIME NOT NULL
);

-- Conversation history
CREATE TABLE conversations (
    id              TEXT PRIMARY KEY,
    started_at      DATETIME NOT NULL,
    title           TEXT,
    message_count   INTEGER DEFAULT 0
);

CREATE TABLE messages (
    id              TEXT PRIMARY KEY,
    conversation_id TEXT NOT NULL REFERENCES conversations(id),
    role            TEXT NOT NULL,   -- 'user' | 'assistant'
    content         TEXT NOT NULL,
    created_at      DATETIME NOT NULL,
    cited_docs      TEXT             -- JSON: list of doc IDs cited
);

-- FTS5 for keyword search over all documents
CREATE VIRTUAL TABLE documents_fts USING fts5(
    title, clean_content,
    content='documents',
    content_rowid='rowid',
    tokenize='porter unicode61'
);
🧠 The Hybrid Retrieval System
Python

# memory/retriever.py

class HybridRetriever:
    """
    Three-layer retrieval system:
    1. Semantic (vector similarity via ChromaDB)
    2. Keyword (BM25 via FTS5)
    3. Graph (traverse knowledge graph for related entities)
    Combined with temporal re-ranking.
    """

    def retrieve(self, query: str, query_meta: QueryMeta) -> List[MemoryResult]:

        # Layer 1: Vector search
        embedding = self.embedder.embed(query)
        semantic_results = self.chroma.query(
            query_embeddings=[embedding],
            n_results=20,
            where=self._build_time_filter(query_meta)
        )

        # Layer 2: BM25 keyword search
        keyword_results = self.db.execute("""
            SELECT d.*, rank
            FROM documents_fts
            JOIN documents d ON documents_fts.rowid = d.rowid
            WHERE documents_fts MATCH ?
            ORDER BY rank LIMIT 20
        """, [query])

        # Layer 3: Graph traversal
        # Extract entities from query
        query_entities = self.entity_extractor.extract(query)
        graph_results = []
        for entity in query_entities:
            # Find all documents connected to this entity in the graph
            connected = self.graph.find_connected_documents(
                entity_name=entity.name,
                hops=2  # traverse up to 2 relationship hops
            )
            graph_results.extend(connected)

        # Merge + re-rank
        merged = self.ranker.merge_and_rank(
            semantic=semantic_results,
            keyword=keyword_results,
            graph=graph_results,
            temporal_preference=query_meta.time_reference,
            limit=10
        )

        return merged
💬 Conversation Interface
text

┌─────────────────────────── MEMEX ────────────────────────────┐
│  Your living memory  │  12,847 documents  │  4.2M words       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  YOU: What was that idea I had about using Bloom filters     │
│       for deduplication back when I was working on the       │
│       distributed cache thing?                              │
│                                                              │
│  MEMEX: You explored this in March 2024. In a note titled    │
│  "cache_dedup_ideas.md" [📄], you sketched using a Bloom    │
│  filter with a 1% false positive rate to avoid duplicate    │
│  cache writes. You also referenced a paper [📄 "Bloom       │
│  Filters in Distributed Systems" — saved Feb 2024] and      │
│  linked it to a Slack thread with @james about Redis        │
│  memory pressure [📧 Mar 12].                               │
│                                                              │
│  Three days later you opened a similar Stack Overflow        │
│  answer [🌐] and starred it — want me to pull that up too?  │
│                                                              │
│  YOU: Yes, and show me everything related to that project    │
│                                                              │
│  MEMEX: Here's the full memory cluster for "distributed      │
│  cache project" spanning Feb–May 2024: ...                  │
│                                                              │
│  [Type your question...                              ] [⏎]  │
└──────────────────────────────────────────────────────────────┘
📦 Tech Stack Summary
Layer	Technology
Language	Python 3.11+
Local LLM	Ollama (llama3, mistral)
Embedding	nomic-embed-text via Ollama
Vector DB	ChromaDB (local)
Graph DB	KuzuDB (embedded graph DB)
Full-Text Search	SQLite FTS5 + BM25
Event Store	SQLite
PDF Parsing	pdfminer.six
Web Parsing	readability-lxml
Code Parsing	tree-sitter
OCR	Tesseract
Browser Extension	TypeScript + MV3
TUI	Textual
Web UI	Svelte + D3.js (knowledge graph)
File Watching	watchdog
🚀 Build Phases
text

Phase 1 — Core Ingestion (4 weeks)
├── Browser extension (captures visited pages)
├── File system watcher (PDF, MD, code)
├── Universal content parser
└── SQLite document store

Phase 2 — Embedding + Vector Search (3 weeks)
├── Ollama integration (nomic-embed-text)
├── ChromaDB local setup
├── Smart chunking strategy
└── Semantic search endpoint

Phase 3 — Knowledge Graph (3 weeks)
├── Entity extraction (local NER via spaCy)
├── Relation extraction (local LLM)
├── KuzuDB graph building
└── Graph traversal API

Phase 4 — Conversation Interface (2 weeks)
├── Hybrid retrieval system
├── Context builder
├── LLM chat with citations
└── Textual TUI chat screen

Phase 5 — Web UI + Polish (3 weeks)
├── Svelte frontend
├── D3.js knowledge graph visualization
├── Timeline browser
└── Terminal/email ingestors
