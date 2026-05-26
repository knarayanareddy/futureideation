MirrorMind
A local AI that watches how you think — not what you do — and builds a living map of your cognitive patterns
🧭 Vision & Core Philosophy
MirrorMind is the most ambitious and philosophical of all five projects. It is fundamentally different from everything else on this list.

Every other tool in this list captures what you do. MirrorMind captures how you think.

It passively reads your writing — notes, journal entries, emails, documents, commit messages, chat logs, anything you give it — and it doesn't summarize them or search them. Instead, it runs a continuous cognitive cartography process: building a living, evolving map of your thinking patterns, cognitive biases, reasoning styles, mental models, and intellectual blind spots.

Over time it builds reports like:

"You tend to reach for analogies when explaining technical ideas — 73% of your explanations use an analogy structure." "You make decisions under uncertainty using loss aversion framing 61% of the time." "You have a recurring blind spot around second-order effects — you rarely explore consequences of consequences." "Your writing shows that when you feel stressed, your sentences get 40% shorter and you use more absolute language." "You've mentioned 'distributed systems' 847 times over 3 years but your thinking around it clusters into 3 distinct mental models that haven't evolved since 2022."

It is a personal cognitive observatory — not therapy, not productivity, not journaling. It is applied metacognition: a tool for understanding the shape of your own mind, locally, privately, and with ruthless honesty.

🏗️ System Architecture
text

┌──────────────────────────────────────────────────────────────┐
│                      MirrorMind                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Content Ingestion                       │    │
│  │  (Notes, journals, emails, docs, commits, chats)     │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │           Cognitive Feature Extractor                │    │
│  │                                                      │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │    │
│  │  │Reasoning │ │Cognitive │ │Linguistic│ │Emotion │  │    │
│  │  │Structure │ │Bias      │ │Pattern   │ │State   │  │    │
│  │  │Detector  │ │Detector  │ │Analyzer  │ │Detector│  │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘  │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐  │    │
│  │  │Mental    │ │Certainty │ │  Decision Style      │  │    │
│  │  │Model     │ │Hedging   │ │  Analyzer            │  │    │
│  │  │Tracker   │ │Detector  │ │                      │  │    │
│  │  └──────────┘ └──────────┘ └──────────────────────┘  │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │           Local LLM Analysis Engine                  │    │
│  │           (Ollama: mistral / llama3)                 │    │
│  │  Deep analysis of reasoning structures + patterns    │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Cognitive Graph                         │    │
│  │  (Concepts + Mental Models + Their Evolution)        │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Insight Engine                          │    │
│  │  (Generates weekly/monthly cognitive reports)        │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              UI (Svelte + D3.js mind map)            │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
🗂️ Full Directory Structure
text

mirrormind/
│
├── mirrormind/                      # Python package
│   ├── __init__.py
│   ├── main.py
│   ├── config.py
│   │
│   ├── ingestors/
│   │   ├── base.py
│   │   ├── obsidian.py              # Obsidian vault reader
│   │   ├── logseq.py                # Logseq graph reader
│   │   ├── plaintext.py             # Plain text/markdown files
│   │   ├── email.py                 # IMAP email reader
│   │   ├── git_commits.py           # Git commit messages
│   │   └── chat.py                  # Exported chat logs
│   │
│   ├── extractors/
│   │   ├── __init__.py
│   │   ├── reasoning/
│   │   │   ├── structure.py         # Detect reasoning type:
│   │   │   │   # deductive | inductive | abductive | analogical | causal
│   │   │   ├── argument.py          # Claim-evidence-warrant structure
│   │   │   ├── counterarg.py        # Do you consider counter-arguments?
│   │   │   └── conclusion.py        # How do you reach conclusions?
│   │   │
│   │   ├── biases/
│   │   │   ├── __init__.py
│   │   │   ├── catalog.py           # Full cognitive bias catalog (200+)
│   │   │   ├── confirmation.py      # Confirmation bias detector
│   │   │   ├── availability.py      # Availability heuristic
│   │   │   ├── dunning_kruger.py    # Overconfidence + underconfidence
│   │   │   ├── sunk_cost.py         # Sunk cost fallacy
│   │   │   ├── anchoring.py         # Anchoring bias
│   │   │   ├── attribution.py       # Attribution errors
│   │   │   └── black_white.py       # Black-and-white thinking
│   │   │
│   │   ├── linguistic/
│   │   │   ├── certainty.py         # Hedging vs. absolute language
│   │   │   ├── complexity.py        # Sentence + concept complexity over time
│   │   │   ├── analogy.py           # Analogy usage frequency + quality
│   │   │   ├── abstraction.py       # Abstraction level tracker
│   │   │   └── emotional_tone.py    # Emotional coloring of writing
│   │   │
│   │   ├── mental_models/
│   │   │   ├── detector.py          # Detect which mental models you use
│   │   │   ├── catalog.py           # 100+ named mental models
│   │   │   ├── evolution.py         # Track how models evolve over time
│   │   │   └── gaps.py              # Which models are you never using?
│   │   │
│   │   └── decision/
│   │       ├── style.py             # Fast vs slow thinking preference
│   │       ├── framing.py           # Gain vs loss framing
│   │       ├── uncertainty.py       # How you handle uncertainty
│   │       └── reversibility.py     # Two-way vs one-way door decisions
│   │
│   ├── llm/
│   │   ├── analyzer.py              # Deep LLM-based passage analysis
│   │   ├── insight_generator.py     # Generate insights from patterns
│   │   ├── question_generator.py    # Generate Socratic questions
│   │   └── ollama.py                # Ollama interface
│   │
│   ├── graph/
│   │   ├── cognitive_graph.py       # Concept + mental model graph
│   │   ├── evolution_tracker.py     # Track how concepts evolve
│   │   └── cluster.py               # Cluster related thinking patterns
│   │
│   ├── reports/
│   │   ├── weekly.py                # Weekly cognitive report generator
│   │   ├── monthly.py               # Monthly deep report
│   │   ├── renderer.py              # Markdown/HTML report renderer
│   │   └── templates/
│   │       ├── weekly_report.md.j2
│   │       └── monthly_report.md.j2
│   │
│   ├── db/
│   │   ├── database.py
│   │   └── schema.sql
│   │
│   └── ui/
│       ├── api/
│       │   ├── server.py            # FastAPI
│       │   └── routes/
│       │       ├── patterns.py
│       │       ├── reports.py
│       │       ├── graph.py
│       │       └── ask.py           # Ask MirrorMind a question about yourself
│       └── web/
│           └── src/
│               ├── routes/
│               │   ├── +page.svelte         # Dashboard overview
│               │   ├── patterns/+page.svelte # Cognitive patterns
│               │   ├── biases/+page.svelte  # Bias report
│               │   ├── models/+page.svelte  # Mental model map
│               │   ├── reports/+page.svelte # Weekly/monthly reports
│               │   └── ask/+page.svelte     # Ask about yourself
│               └── components/
│                   ├── CognitiveDNA.svelte  # Your cognitive fingerprint
│                   ├── BiasRadar.svelte     # Radar chart of biases
│                   ├── MindMap.svelte       # D3.js mental model network
│                   ├── EvolutionChart.svelte # How thinking changes over time
│                   ├── PatternCard.svelte
│                   └── InsightFeed.svelte
│
├── pyproject.toml
├── Makefile
└── README.md
🗄️ Database Schema
SQL

-- All ingested writing samples
CREATE TABLE writings (
    id              TEXT PRIMARY KEY,
    source_type     TEXT NOT NULL,   -- 'note' | 'journal' | 'email' | 'commit'
    source_path     TEXT,
    title           TEXT,
    content         TEXT NOT NULL,
    word_count      INTEGER,
    written_at      DATETIME,
    ingested_at     DATETIME NOT NULL,
    checksum        TEXT UNIQUE,
    is_analyzed     BOOLEAN DEFAULT FALSE
);

-- Cognitive patterns detected in writings
CREATE TABLE cognitive_patterns (
    id              TEXT PRIMARY KEY,
    writing_id      TEXT NOT NULL REFERENCES writings(id),
    detected_at     DATETIME NOT NULL,
    pattern_type    TEXT NOT NULL,
    -- 'reasoning_structure' | 'cognitive_bias' | 'mental_model' |
    -- 'linguistic_pattern' | 'decision_style' | 'emotional_state'
    pattern_name    TEXT NOT NULL,   -- e.g. "confirmation_bias", "analogical_reasoning"
    confidence      REAL NOT NULL,
    evidence        TEXT NOT NULL,   -- Quote from writing that shows this pattern
    explanation     TEXT NOT NULL,   -- Why this was flagged
    severity        TEXT,            -- For biases: 'mild' | 'moderate' | 'strong'
    is_positive     BOOLEAN          -- Is this a strength or blind spot?
);

-- Aggregate statistics per pattern (rolling)
CREATE TABLE pattern_stats (
    pattern_name    TEXT PRIMARY KEY,
    pattern_type    TEXT NOT NULL,
    total_count     INTEGER DEFAULT 0,
    frequency_pct   REAL,            -- What % of writings show this
    first_seen      DATETIME,
    last_seen       DATETIME,
    trend           TEXT,            -- 'increasing' | 'decreasing' | 'stable'
    trend_delta     REAL             -- Rate of change per month
);

-- Mental models you use (named models from Charlie Munger's latticework etc.)
CREATE TABLE mental_models (
    id              TEXT PRIMARY KEY,
    name            TEXT UNIQUE NOT NULL,   -- e.g. "First Principles", "Inversion"
    category        TEXT,                   -- e.g. "Physics", "Economics", "Psychology"
    description     TEXT,
    usage_count     INTEGER DEFAULT 0,
    first_used      DATETIME,
    last_used       DATETIME,
    evolution_notes TEXT                    -- How your use of this model has changed
);

-- Concept evolution tracking
CREATE TABLE concept_history (
    id              TEXT PRIMARY KEY,
    concept         TEXT NOT NULL,
    writing_id      TEXT NOT NULL REFERENCES writings(id),
    recorded_at     DATETIME NOT NULL,
    framing         TEXT,            -- How you framed this concept this time
    embedding       BLOB,            -- Vector embedding for semantic drift tracking
    sentiment       REAL             -- Emotional valence toward this concept
);

-- Generated insights and reports
CREATE TABLE insights (
    id              TEXT PRIMARY KEY,
    generated_at    DATETIME NOT NULL,
    insight_type    TEXT,            -- 'weekly' | 'monthly' | 'triggered' | 'ad_hoc'
    title           TEXT NOT NULL,
    content_md      TEXT NOT NULL,   -- Full report in Markdown
    pattern_refs    TEXT,            -- JSON: patterns referenced in this insight
    user_notes      TEXT             -- User's response/reflection on insight
);

-- Questions MirrorMind asks you (Socratic method)
CREATE TABLE questions (
    id              TEXT PRIMARY KEY,
    generated_at    DATETIME NOT NULL,
    question        TEXT NOT NULL,
    context         TEXT,            -- What pattern triggered this question
    user_answer     TEXT,
    answered_at     DATETIME
);
🧠 Sample Cognitive Report Output
Markdown

# 🔮 MirrorMind Weekly Report — Week 3, January 2025

## Your Cognitive Fingerprint This Week

You wrote 4,231 words across 12 documents.
MirrorMind analyzed 47 cognitive patterns.

---

## 💡 REASONING STYLE (This week)

Your dominant reasoning style: **Analogical (68%)**
You reached for analogies in 8 of 12 documents.
This is a strength — analogies accelerate understanding.
**Watch for**: analogies that map imperfectly but feel convincing.

---

## 🧠 MENTAL MODELS IN USE

✅ **First Principles** (3 uses) — stronger than last month
✅ **Inversion** (2 uses) — new this week!
⚠️ **Second-Order Thinking** — 0 uses this week, 3rd consecutive week

> MirrorMind: You haven't explored second-order effects in 3 weeks.
> Your last document about deployment strategy stopped at first-order
> consequences. What happens if THAT outcome is also wrong?

---

## 🎯 COGNITIVE BIASES DETECTED

🟡 **Confirmation Bias** (moderate) — seen in 4/12 documents
   Evidence: "This confirms my earlier thinking that..."
   You searched for evidence supporting your existing view
   without exploring contradictions.

🟠 **Planning Fallacy** (strong) — in your project timeline note
   Evidence: "This should take about 2 weeks..."
   You consistently underestimate by ~40% based on your history.

---

## 📊 YOUR CERTAINTY SPECTRUM

Absolute language this week:  ████████░░  78% (↑ from 61% last week)
Hedging language this week:   ██░░░░░░░░  22%

> MirrorMind: Your language became more absolute this week.
> This coincides with your writing about [project X].
> Are you more certain, or less open to being wrong?

---

## ❓ THIS WEEK'S QUESTION FOR YOU

"In your note about the infrastructure decision — you wrote that
option A was 'clearly better'. What would have to be true for
option B to be the right answer?"
🖥️ Dashboard UI
text

┌─────────────────────── MirrorMind ───────────────────────────┐
│  Your mind, mapped. Locally. Privately. Honestly.            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  COGNITIVE DNA                                               │
│  ─────────────                                               │
│  Reasoning:    Analogical  ████████░░  Primary style         │
│  Decision:     Systematic  ██████░░░░  Slow thinking pref.   │
│  Certainty:    High        ████████░░  Tends toward absolutes│
│  Complexity:   High        ██████████  Dense, multi-layered  │
│  Emotion:      Analytical  ███░░░░░░░  Low emotional coloring│
│                                                              │
│  TOP MENTAL MODELS YOU USE:                                  │
│  1. First Principles    ████████████  (used 47 times)        │
│  2. Occam's Razor       ██████████░░  (used 38 times)        │
│  3. Inversion           ████████░░░░  (used 31 times)        │
│                                                              │
│  MODELS YOU NEVER USE (blind spots):                         │
│  • Probabilistic Thinking   • Regression to Mean            │
│  • Availability Heuristic awareness                         │
│                                                              │
│  LATEST INSIGHT:                                             │
│  "Your concept of 'scalability' has shifted significantly    │
│   over 18 months — from infrastructure-focused to team      │
│   process-focused. Want to see the evolution?"              │
│                                                              │
│  [View Full Report] [Ask MirrorMind] [Mental Model Map]     │
└──────────────────────────────────────────────────────────────┘
📦 Tech Stack Summary
Layer	Technology
Language	Python 3.11+
Local LLM	Ollama (mistral / llama3 / phi3)
Embedding	nomic-embed-text (via Ollama)
NLP	spaCy + custom rule engines
Database	SQLite + FTS5
Graph	NetworkX + KuzuDB
Report Generation	Jinja2 templating
Web UI	FastAPI + Svelte + D3.js
Visualization	D3.js (mind map, radar, timeline)
🚀 Build Phases
text

Phase 1 — Ingestion + Basic Extraction (3 weeks)
├── Obsidian / plaintext reader
├── Reasoning structure detector
├── Certainty + hedging analyzer
└── SQLite schema + storage

Phase 2 — Bias + Mental Model Detectors (4 weeks)
├── Top 20 cognitive bias detectors
├── Mental model classifier (100 models)
├── Decision style analyzer
└── Pattern statistics engine

Phase 3 — LLM Deep Analysis (2 weeks)
├── Ollama integration
├── Per-document LLM analysis
├── Insight generation
└── Socratic question generator

Phase 4 — Cognitive Graph (2 weeks)
├── Concept tracking over time
├── Mental model evolution
├── Semantic drift detection
└── Blind spot identification

Phase 5 — Reports + Dashboard (3 weeks)
├── Weekly/monthly report generator
├── Svelte dashboard
├── D3.js mind map + radar chart
└── "Ask MirrorMind" interface
