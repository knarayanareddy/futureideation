ChronoMap
A living, self-building visual map of YOUR knowledge — built entirely from your files, notes, and browsing — that grows like a brain
🧭 The Vision
Every file you write, every article you save, every PDF you annotate, every bookmark you make, every note you take — ChronoMap reads all of it and builds a beautiful, interactive, local-first knowledge graph that looks like a glowing neural map of your brain. Nodes are concepts, ideas, people, projects. Edges are relationships between them. The graph evolves in real-time as you learn new things. You can zoom in to see a topic cluster, zoom out to see your entire intellectual life, travel back in time to see what your knowledge graph looked like 6 months ago, and search it semantically. It reveals things about how you think that you could never see yourself — your intellectual blind spots, your obsessions, your forgotten ideas, your conceptual islands (topics you know about but never connected). It is a mirror of your mind. No cloud. No AI company. Just you, your data, and the map.

🏗️ Why It Doesn't Exist Yet
Obsidian does local knowledge graphs — but only for Markdown notes you manually link. Roam Research does similar. LogSeq too. But none of them:

Automatically ingest ALL your data sources (PDFs, emails, browser bookmarks, browser history, code files, calendar, ebook highlights)
Use local NLP to auto-extract entities, concepts, and relationships (no manual linking)
Show temporal evolution of your knowledge (time-travel the graph)
Identify intellectual blind spots (concepts that should be connected but aren't)
Use semantic similarity to surface forgotten connections
Work as a passive background process that updates without you doing anything
🗂️ Full Directory Structure
text

chronomap/
│
├── ingestors/                     # Data source readers
│   ├── markdown_ingestor.py       # Obsidian/Roam/LogSeq notes
│   ├── pdf_ingestor.py            # PDFs + highlights (PyMuPDF)
│   ├── browser_ingestor.py        # Chrome/Firefox bookmarks + history
│   ├── email_ingestor.py          # Local maildir / mbox files
│   ├── code_ingestor.py           # Source code: extracts concepts from comments
│   ├── ebook_ingestor.py          # Kindle/Kobo highlights (via export)
│   ├── calendar_ingestor.py       # .ics files
│   └── watch_daemon.py            # inotify watcher: re-ingest on file change
│
├── extractor/                     # NLP entity + relationship extraction
│   ├── entity_extractor.py        # spaCy NER: people, orgs, concepts, places
│   ├── concept_extractor.py       # KeyBERT keyword extraction
│   ├── relation_extractor.py      # Dependency parsing → relationships
│   ├── embedder.py                # Sentence embeddings (all-MiniLM-L6-v2)
│   └── topic_modeler.py           # BERTopic: cluster documents into topics
│
├── graph/
│   ├── builder.py                 # Builds graph from extracted entities
│   ├── merger.py                  # Deduplicates nodes (same concept, diff names)
│   ├── scorer.py                  # Edge weight = co-occurrence + semantic sim
│   ├── temporal.py                # Timestamps all nodes/edges for time travel
│   ├── blind_spot_detector.py     # Finds concept clusters that should connect
│   └── island_detector.py         # Finds isolated knowledge islands
│
├── db/
│   ├── graph_db.py                # NetworkX graph + SQLite persistence
│   ├── vector_store.py            # sqlite-vss or Chroma for semantic search
│   └── schema.sql
│
├── api/
│   ├── server.py                  # FastAPI: serve graph data to frontend
│   └── routes/
│       ├── graph.py               # Full graph, subgraph, node detail
│       ├── search.py              # Semantic + keyword search
│       ├── temporal.py            # Time-travel: graph at timestamp T
│       └── insights.py            # Blind spots, islands, obsessions
│
└── ui/                            # Svelte + D3.js / Three.js frontend
    ├── src/
    │   ├── components/
    │   │   ├── GraphCanvas.svelte      # Main 3D force-directed graph
    │   │   ├── TimeSlider.svelte       # Scrub back in time
    │   │   ├── NodeDetail.svelte       # Click a node → see all sources
    │   │   ├── BlindSpotPanel.svelte   # "You know X and Y but never linked them"
    │   │   ├── SearchBar.svelte        # Semantic search
    │   │   ├── ClusterView.svelte      # Zoom into topic clusters
    │   │   └── ObsessionTracker.svelte # Your most-referenced concepts over time
    │   └── lib/
    │       ├── graph-renderer.ts       # Three.js 3D graph rendering
    │       ├── force-layout.ts         # D3 force simulation
    │       └── time-travel.ts          # Temporal graph interpolation
    └── package.json
🧠 Key Intelligence Modules
Python

# blind_spot_detector.py
# Finds concepts that SHOULD be connected but aren't

class BlindSpotDetector:
    def detect(self, graph: KnowledgeGraph) -> List[BlindSpot]:
        blind_spots = []

        for node_a in graph.nodes():
            for node_b in graph.nodes():
                if node_a == node_b:
                    continue

                # Are these semantically similar?
                sim = cosine_similarity(
                    node_a.embedding,
                    node_b.embedding
                )

                # But NOT connected in the graph?
                path_length = graph.shortest_path_length(node_a, node_b)

                if sim > 0.75 and path_length > 4:
                    blind_spots.append(BlindSpot(
                        node_a=node_a,
                        node_b=node_b,
                        similarity=sim,
                        graph_distance=path_length,
                        message=f"You know a lot about '{node_a.label}' "
                                f"and '{node_b.label}' separately — "
                                f"but you've never connected them."
                    ))

        return sorted(blind_spots, key=lambda x: x.similarity, reverse=True)
Python

# temporal.py
# Time-travel: what did my knowledge graph look like 6 months ago?

class TemporalGraph:
    def graph_at(self, timestamp: datetime) -> KnowledgeGraph:
        """
        Returns a snapshot of the knowledge graph
        as it existed at a given point in time.
        All nodes/edges have first_seen and last_seen timestamps.
        Filters to only include entities that existed at T.
        """
        filtered_nodes = [
            n for n in self.all_nodes
            if n.first_seen <= timestamp
        ]
        filtered_edges = [
            e for e in self.all_edges
            if e.first_seen <= timestamp
        ]
        return KnowledgeGraph(nodes=filtered_nodes, edges=filtered_edges)
🎨 UI Design
text

┌──────────────────── ChronoMap ────────────────────────────┐
│  🔍 [Search your knowledge...              ]   [⏮ Time]  │
│                                                           │
│          ·  machine learning                              │
│       ·     ·   ·                                         │
│     ·  AI  ·  neural networks ·                           │
│    ·     ·     ·        ·    ·                            │
│   · cognition ·   ·  · attention ·                        │
│    ·    ·   philosophy ·   ·    ·                         │
│      ·    ·   ·  consciousness ·                          │
│        ·    ethics  ·    ·                                │
│           ·    ·   ·                                      │
│              privacy ←──── 🔴 BLIND SPOT                  │
│                 ·     (93% similar to                     │
│                        "surveillance" cluster             │
│                         — but unconnected)                │
│                                                           │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━           │
│  ◀ Jan 2024 ──────────────────────── Now ▶               │
└───────────────────────────────────────────────────────────┘
📦 Tech Stack
Layer	Technology
Language	Python + TypeScript
NLP	spaCy, KeyBERT, BERTopic
Embeddings	sentence-transformers (local)
Graph	NetworkX + SQLite
Vector Search	sqlite-vss or Chroma
PDF parsing	PyMuPDF
API	FastAPI
UI	Svelte + Three.js + D3.js
File Watching	watchdog
