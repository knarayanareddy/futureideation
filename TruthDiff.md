TruthDiff
Catches silent edits to news articles
🧭 Vision & Core Philosophy
TruthDiff is a media accountability tool that works silently in the background. The moment a user visits a news article, it snapshots it. It then re-fetches that article on a schedule and alerts the user if anything changed — headlines, body text, author names, publication dates, image captions, even metadata. It presents changes in a beautiful, readable git-diff style UI. The goal: make silent editorial revisionism visible, archivable, and shareable.

🏗️ System Architecture
text

┌─────────────────────────────────────────────────────────────┐
│                    USER'S BROWSER                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │           TruthDiff Browser Extension                │   │
│  │  - Detects news article pages                        │   │
│  │  - Sends article URL + snapshot to backend           │   │
│  │  - Shows diff badge/notification on revisit          │   │
│  │  - Popup UI for viewing diffs                        │   │
│  └────────────────────┬─────────────────────────────────┘   │
└───────────────────────│─────────────────────────────────────┘
                        │ HTTP (localhost or remote)
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                  TruthDiff Backend (Go)                     │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │  REST API   │  │  Scheduler   │  │  Diff Engine      │  │
│  │  Server     │  │  (re-fetch   │  │  (diff-match-     │  │
│  │             │  │   articles)  │  │   patch / go)     │  │
│  └──────┬──────┘  └──────┬───────┘  └────────┬──────────┘  │
│         │                │                    │             │
│         └────────────────┼────────────────────┘             │
│                          ▼                                  │
│              ┌───────────────────────┐                      │
│              │   SQLite Database     │                      │
│              │   (via SQLCipher)     │                      │
│              └───────────────────────┘                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Article Fetcher / Scraper              │    │
│  │   (Colly + Readability + Goquery)                   │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│              TruthDiff Web UI (optional)                    │
│         (Svelte or htmx — self-hosted dashboard)           │
└─────────────────────────────────────────────────────────────┘
🗂️ Full Directory Structure
text

truthdiff/
│
├── backend/                        # Go backend
│   ├── main.go                     # Entry point
│   ├── config/
│   │   └── config.go               # App config (ports, DB path, schedule)
│   ├── api/
│   │   ├── router.go               # Chi router setup
│   │   ├── handlers/
│   │   │   ├── articles.go         # POST /article, GET /articles
│   │   │   ├── diffs.go            # GET /diffs/:id, GET /diffs/article/:url
│   │   │   ├── snapshots.go        # GET /snapshots/:id
│   │   │   └── alerts.go           # GET /alerts, DELETE /alerts/:id
│   ├── db/
│   │   ├── db.go                   # SQLite connection + SQLCipher setup
│   │   ├── migrations/
│   │   │   ├── 001_create_articles.sql
│   │   │   ├── 002_create_snapshots.sql
│   │   │   ├── 003_create_diffs.sql
│   │   │   └── 004_create_alerts.sql
│   │   └── queries/
│   │       ├── articles.go
│   │       ├── snapshots.go
│   │       └── diffs.go
│   ├── scraper/
│   │   ├── fetcher.go              # HTTP fetch with rotating UA strings
│   │   ├── extractor.go            # Readability extraction of article body
│   │   ├── cleaner.go              # Normalize HTML → clean text
│   │   └── metadata.go             # Extract title, author, date, og:tags
│   ├── differ/
│   │   ├── engine.go               # Core diff logic
│   │   ├── text_diff.go            # Word-level and line-level diffs
│   │   ├── metadata_diff.go        # Title/author/date change detection
│   │   └── severity.go             # Score how significant a change is
│   ├── scheduler/
│   │   ├── scheduler.go            # Cron-based re-fetch logic (gocron)
│   │   ├── queue.go                # Article re-check queue
│   │   └── worker.go               # Worker pool for fetching
│   ├── notifier/
│   │   ├── notifier.go             # Notification dispatcher
│   │   ├── desktop.go              # OS desktop notification
│   │   └── webhook.go              # Optional webhook push
│   └── models/
│       ├── article.go
│       ├── snapshot.go
│       ├── diff.go
│       └── alert.go
│
├── extension/                      # Browser Extension (TypeScript)
│   ├── manifest.json               # MV3 manifest
│   ├── src/
│   │   ├── background/
│   │   │   ├── background.ts       # Service worker
│   │   │   ├── detector.ts         # Detects if page is a news article
│   │   │   └── api.ts              # Communicates with Go backend
│   │   ├── content/
│   │   │   ├── content.ts          # Injected into article pages
│   │   │   ├── badge.ts            # Injects "N changes detected" badge on page
│   │   │   └── highlighter.ts      # Highlights changed sections inline
│   │   ├── popup/
│   │   │   ├── popup.html
│   │   │   ├── popup.ts            # Extension popup UI
│   │   │   └── popup.css
│   │   └── shared/
│   │       ├── types.ts
│   │       └── utils.ts
│   ├── icons/
│   └── tsconfig.json
│
├── ui/                             # Self-hosted Web Dashboard (Svelte)
│   ├── src/
│   │   ├── routes/
│   │   │   ├── +page.svelte        # Dashboard home
│   │   │   ├── article/[id]/       # Article detail + all diffs
│   │   │   └── diff/[id]/          # Full diff viewer
│   │   ├── components/
│   │   │   ├── DiffViewer.svelte   # Git-diff style renderer
│   │   │   ├── ArticleCard.svelte
│   │   │   ├── ChangeTimeline.svelte
│   │   │   ├── SeverityBadge.svelte
│   │   │   └── AlertPanel.svelte
│   │   ├── stores/
│   │   │   ├── articles.ts
│   │   │   └── alerts.ts
│   │   └── lib/
│   │       ├── api.ts
│   │       └── diff-renderer.ts    # Renders diff objects as HTML
│   └── package.json
│
├── docker-compose.yml
├── Dockerfile
├── README.md
└── Makefile
🗄️ Database Schema
SQL

-- Articles being watched
CREATE TABLE articles (
    id              TEXT PRIMARY KEY,       -- UUID
    url             TEXT UNIQUE NOT NULL,
    domain          TEXT NOT NULL,          -- e.g. "nytimes.com"
    first_seen_at   DATETIME NOT NULL,
    last_checked_at DATETIME,
    check_interval  INTEGER DEFAULT 3600,   -- seconds
    is_active       BOOLEAN DEFAULT TRUE,
    snapshot_count  INTEGER DEFAULT 0
);

-- Every fetched version of an article
CREATE TABLE snapshots (
    id              TEXT PRIMARY KEY,       -- UUID
    article_id      TEXT NOT NULL REFERENCES articles(id),
    fetched_at      DATETIME NOT NULL,
    title           TEXT,
    author          TEXT,
    published_date  TEXT,
    body_text       TEXT NOT NULL,          -- Cleaned plain text
    body_html       TEXT,                   -- Raw HTML (compressed)
    word_count      INTEGER,
    checksum        TEXT NOT NULL,          -- SHA256 of body_text
    metadata_json   TEXT,                   -- og:tags, schema.org, etc.
    http_status     INTEGER,
    fetch_duration  INTEGER                 -- ms
);

-- Detected differences between two snapshots
CREATE TABLE diffs (
    id              TEXT PRIMARY KEY,
    article_id      TEXT NOT NULL REFERENCES articles(id),
    snapshot_a_id   TEXT NOT NULL REFERENCES snapshots(id),   -- older
    snapshot_b_id   TEXT NOT NULL REFERENCES snapshots(id),   -- newer
    detected_at     DATETIME NOT NULL,
    severity        TEXT CHECK(severity IN ('minor','moderate','major')),
    title_changed   BOOLEAN DEFAULT FALSE,
    author_changed  BOOLEAN DEFAULT FALSE,
    date_changed    BOOLEAN DEFAULT FALSE,
    body_changed    BOOLEAN DEFAULT FALSE,
    diff_json       TEXT NOT NULL,          -- Full diff-match-patch output
    summary         TEXT,                   -- Human-readable change summary
    words_added     INTEGER DEFAULT 0,
    words_removed   INTEGER DEFAULT 0
);

-- User-facing alerts
CREATE TABLE alerts (
    id              TEXT PRIMARY KEY,
    diff_id         TEXT NOT NULL REFERENCES diffs(id),
    article_id      TEXT NOT NULL REFERENCES articles(id),
    created_at      DATETIME NOT NULL,
    is_read         BOOLEAN DEFAULT FALSE,
    notification_sent BOOLEAN DEFAULT FALSE
);

-- Domains and their scraping rules
CREATE TABLE domain_rules (
    domain          TEXT PRIMARY KEY,
    content_selector TEXT,                  -- CSS selector for article body
    title_selector  TEXT,
    author_selector TEXT,
    date_selector   TEXT,
    requires_js     BOOLEAN DEFAULT FALSE,
    scrape_delay_ms INTEGER DEFAULT 1000
);
🔌 REST API Endpoints
text

POST   /api/v1/articles              → Submit a new article URL to watch
GET    /api/v1/articles              → List all watched articles
GET    /api/v1/articles/:id          → Get article details
DELETE /api/v1/articles/:id          → Stop watching an article

GET    /api/v1/articles/:id/snapshots          → All snapshots for article
GET    /api/v1/snapshots/:id                   → Single snapshot

GET    /api/v1/articles/:id/diffs              → All diffs for article
GET    /api/v1/diffs/:id                       → Full diff detail
GET    /api/v1/diffs/:id/render                → HTML-rendered diff

GET    /api/v1/alerts                          → Unread alerts
PUT    /api/v1/alerts/:id/read                 → Mark as read
DELETE /api/v1/alerts/:id                      → Dismiss alert

GET    /api/v1/stats                           → Total articles, diffs, domains
GET    /api/v1/export/:article_id              → Export all snapshots as ZIP
⚙️ Core Module Breakdown
1. 📥 Article Fetcher (scraper/fetcher.go)
Go

type FetchResult struct {
    URL          string
    StatusCode   int
    RawHTML      string
    FetchedAt    time.Time
    DurationMS   int64
}

type Fetcher struct {
    client      *http.Client
    userAgents  []string        // Rotating UA pool
    rateLimiter *rate.Limiter   // Per-domain rate limiting
}

// Core fetch logic:
// 1. Check domain_rules for special handling
// 2. Rotate User-Agent header
// 3. Respect per-domain rate limits
// 4. Handle redirects and canonical URLs
// 5. Detect paywalls and flag them
func (f *Fetcher) Fetch(url string) (*FetchResult, error)
2. 📄 Content Extractor (scraper/extractor.go)
Go

type ExtractedArticle struct {
    Title         string
    Author        string
    PublishedDate string
    BodyText      string    // Clean plain text only
    BodyHTML      string    // Sanitized HTML
    WordCount     int
    Metadata      map[string]string
    Checksum      string    // SHA256(BodyText)
}

// Extraction pipeline:
// 1. Try domain-specific CSS selectors first
// 2. Fall back to go-readability
// 3. Strip ads, nav, footer, cookie banners
// 4. Extract schema.org/og: metadata
// 5. Normalize whitespace and Unicode
func Extract(html string, domain string) (*ExtractedArticle, error)
3. 🔍 Diff Engine (differ/engine.go)
Go

type DiffResult struct {
    TitleChanged   bool
    AuthorChanged  bool
    DateChanged    bool
    BodyChanged    bool
    WordsAdded     int
    WordsRemoved   int
    Severity       string          // "minor" | "moderate" | "major"
    Patches        []DiffPatch     // diff-match-patch output
    Summary        string
}

type DiffPatch struct {
    Type    string   // "insert" | "delete" | "equal"
    Text    string
    Offset  int
}

// Diff pipeline:
// 1. Compare checksums first (fast path — no change)
// 2. Diff metadata fields (title, author, date)
// 3. Word-level diff on body text
// 4. Sentence-level diff for context
// 5. Score severity based on:
//    - % of article changed
//    - Headline changes = always MAJOR
//    - Author changes = MAJOR
//    - <5% word change = minor
func Diff(a, b *ExtractedArticle) (*DiffResult, error)
4. ⏱️ Scheduler (scheduler/scheduler.go)
Go

// Re-check schedule strategy:
// - Articles < 1 hour old:     check every 15 mins
// - Articles 1-24 hours old:   check every 1 hour
// - Articles 1-7 days old:     check every 6 hours
// - Articles > 7 days old:     check every 24 hours
// - Articles > 30 days old:    check every 72 hours
// - User can pin any article to a custom interval

type Scheduler struct {
    cron    *gocron.Scheduler
    queue   chan string           // Article IDs to re-check
    workers int
    db      *Database
    fetcher *Fetcher
    differ  *DiffEngine
}
🖥️ Browser Extension Flow
text

User visits nytimes.com/article-xyz
          │
          ▼
content.ts detects: is this a news article?
(checks domain list + og:type="article" + article tag presence)
          │
          ▼
Sends URL to background.ts
          │
          ▼
background.ts calls POST /api/v1/articles
Backend: fetch + snapshot + store
          │
          ▼
On FUTURE visits to same URL:
background.ts calls GET /api/v1/diffs?url=...
          │
    ┌─────┴──────┐
    │            │
No diffs      Diffs found
    │            │
(do nothing)  badge.ts injects "⚠️ 3 changes detected"
              banner into the page
              highlighter.ts underlines changed paragraphs
              popup shows full diff on click
🎨 Diff Viewer UI Design
text

┌────────────────────────────────────────────────────────┐
│  TruthDiff — New York Times                      🔔 3  │
├────────────────────────────────────────────────────────┤
│  Article: "Ukraine Peace Talks Begin in Vienna"        │
│  First seen: Jan 12, 2025 09:14 AM                     │
│  Changes detected: 3 times                             │
├────────────────────────────────────────────────────────┤
│  CHANGE #3 — Jan 12, 2025 4:47 PM      [MAJOR] 🔴     │
│  ─────────────────────────────────────────────────     │
│  HEADLINE CHANGED:                                     │
│  ━━━━━━━━━━━━━━━━                                      │
│  - "Ukraine Peace Talks Begin in Vienna"               │
│  + "Ukraine Peace Talks Collapse in Vienna"            │
│                                                        │
│  BODY CHANGES (+12 words, -47 words):                  │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━                  │
│  ...diplomats confirmed that [-both sides had agreed-] │
│  [+negotiations broke down after-] the meeting...      │
│                                                        │
│  [-"We are optimistic," said the lead negotiator.-]    │
│                                                        │
│  [Share this diff] [Export] [View raw snapshots]       │
└────────────────────────────────────────────────────────┘
📦 Tech Stack Summary
Layer	Technology
Backend	Go 1.22
HTTP Router	Chi
Database	SQLite + SQLCipher
Scraping	Colly + go-readability + goquery
Diffing	go-diff (diff-match-patch port)
Scheduling	gocron
Extension	TypeScript + Vite + MV3
Frontend UI	Svelte 5 + TailwindCSS
Notifications	beeep (cross-platform desktop)
Containerization	Docker + docker-compose
🚀 Build Phases
text

Phase 1 — Core (4 weeks)
├── Go backend scaffold
├── SQLite schema + migrations
├── Article fetcher + extractor
├── Basic diff engine
└── REST API (articles + snapshots + diffs)

Phase 2 — Extension (2 weeks)
├── MV3 browser extension
├── Article detection logic
├── Badge + inline highlighter
└── Popup diff viewer

Phase 3 — Scheduler + Alerts (2 weeks)
├── gocron-based re-fetch scheduler
├── Alert generation
├── Desktop notifications
└── Domain rules system

Phase 4 — Dashboard UI (3 weeks)
├── Svelte dashboard
├── Diff viewer component
├── Timeline view
└── Export functionality

Phase 5 — Polish (2 weeks)
├── Domain-specific scraper rules
├── Severity scoring tuning
├── Shareable diff permalinks
└── Docker packaging
