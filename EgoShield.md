EgoShield
An AI that monitors every digital interaction you have and quietly tells you when you're being psychologically manipulated
🧭 Vision & Core Philosophy
EgoShield is a personal AI sentinel that sits silently in the background and watches how the digital world is trying to manipulate you — in real time. Not just ads and trackers. Deeper than that.

It reads the emails in your inbox, the articles on your screen, the social media posts you scroll through — and flags manipulation tactics as they happen: urgency manufacturing, false scarcity, social proof exploitation, fear induction, reciprocity traps, authority manufacturing, gaslighting patterns, dark UX patterns, emotional hijacking.

It doesn't block anything. It doesn't censor. It just holds up a mirror and says:

"This email is using urgency language and loss aversion in its subject line." "This article uses 11 emotionally loaded words designed to induce outrage." "This checkout page has 4 dark patterns: fake countdown timer, false scarcity badge, pre-ticked upsell, and guilt-trip opt-out."

It's a cognitive firewall — not for your network, but for your mind.

🏗️ System Architecture
text

┌──────────────────────────────────────────────────────────────┐
│                       EgoShield                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Content Capture Layer                   │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────┐  │    │
│  │  │  Browser     │  │  Email       │  │  System   │  │    │
│  │  │  Extension   │  │  Reader      │  │  Notif.   │  │    │
│  │  │  (web pages, │  │  (IMAP local)│  │  Monitor  │  │    │
│  │  │  social feed │  │              │  │           │  │    │
│  │  │  content)    │  │              │  │           │  │    │
│  │  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘  │    │
│  └─────────│────────────────│─────────────────│─────────┘   │
│            └────────────────┼─────────────────┘             │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Content Normalizer                      │    │
│  │  (HTML cleaning, text extraction, structure parsing) │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │           Manipulation Detection Engine              │    │
│  │                                                      │    │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────┐   │    │
│  │  │  Linguistic│ │  Dark UX   │ │  Emotional     │   │    │
│  │  │  Pattern   │ │  Pattern   │ │  Loading       │   │    │
│  │  │  Detector  │ │  Detector  │ │  Analyzer      │   │    │
│  │  └────────────┘ └────────────┘ └────────────────┘   │    │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────┐   │    │
│  │  │  Urgency / │ │  Authority │ │  Social Proof  │   │    │
│  │  │  Scarcity  │ │  Manufact. │ │  Exploiter     │   │    │
│  │  │  Detector  │ │  Detector  │ │  Detector      │   │    │
│  │  └────────────┘ └────────────┘ └────────────────┘   │    │
│  │                                                      │    │
│  │  ┌────────────────────────────────────────────────┐  │    │
│  │  │    Local LLM Arbiter (Ollama / mistral)        │  │    │
│  │  │    (Final classification + explanation)        │  │    │
│  │  └────────────────────────────────────────────────┘  │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Alert + Annotation System               │    │
│  │  (Browser overlay, desktop notification, log)        │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
🗂️ Full Directory Structure
text

egoshield/
│
├── backend/                         # Python backend daemon
│   ├── main.py
│   ├── config.py
│   │
│   ├── detectors/
│   │   ├── __init__.py
│   │   ├── base.py                  # Abstract Detector base class
│   │   │
│   │   ├── linguistic/
│   │   │   ├── urgency.py           # "Act now!" "Limited time!" "Don't miss out!"
│   │   │   ├── scarcity.py          # "Only 3 left!" "Selling fast!"
│   │   │   ├── social_proof.py      # "Thousands of happy customers!"
│   │   │   ├── authority.py         # Fake expertise, credential dropping
│   │   │   ├── reciprocity.py       # "We gave you X, now you owe us Y"
│   │   │   ├── loss_aversion.py     # Framing as loss vs. gain
│   │   │   ├── fear_induction.py    # Fear/threat language patterns
│   │   │   ├── gaslighting.py       # Reality denial, DARVO patterns
│   │   │   └── emotional_loading.py # Emotionally charged word density
│   │   │
│   │   ├── ux/
│   │   │   ├── dark_patterns.py     # Dark UX pattern detector (visual heuristics)
│   │   │   ├── countdown.py         # Fake countdown timer detection
│   │   │   ├── pretick.py           # Pre-ticked checkbox detection
│   │   │   ├── confirm_shame.py     # "No thanks, I hate saving money"
│   │   │   └── misdirection.py      # Visual hierarchy manipulation
│   │   │
│   │   ├── content/
│   │   │   ├── outrage_bait.py      # Outrage-optimized article detector
│   │   │   ├── clickbait.py         # Headline clickbait scorer
│   │   │   └── propaganda.py        # Propaganda technique classifier
│   │   │
│   │   └── llm_arbiter.py           # LLM final-pass classification
│   │
│   ├── capture/
│   │   ├── email_reader.py          # IMAP reader (local only)
│   │   └── notification_monitor.py  # OS notification text capture
│   │
│   ├── scoring/
│   │   ├── scorer.py                # Aggregate manipulation score
│   │   ├── severity.py              # LOW / MEDIUM / HIGH / CRITICAL
│   │   └── explainer.py             # Human-readable explanation generator
│   │
│   ├── db/
│   │   ├── database.py
│   │   └── schema.sql
│   │
│   └── api/
│       ├── server.py                # FastAPI server
│       └── routes/
│           ├── analyze.py           # POST /analyze (content → alerts)
│           ├── history.py           # GET /history
│           └── stats.py             # GET /stats (your manipulation exposure)
│
├── extension/                       # Browser Extension (TypeScript)
│   ├── manifest.json
│   └── src/
│       ├── background/
│       │   ├── background.ts        # Page content → backend analysis
│       │   └── analyzer.ts
│       ├── content/
│       │   ├── content.ts           # Injected into pages
│       │   ├── overlay.ts           # Visual annotation overlay
│       │   ├── highlighter.ts       # Highlight manipulative text
│       │   └── darkpattern.ts       # Detect + highlight dark UX elements
│       └── popup/
│           ├── popup.html           # Extension popup: page score
│           └── popup.ts
│
├── ui/                              # Svelte dashboard
│   └── src/
│       ├── routes/
│       │   ├── +page.svelte         # Dashboard: today's manipulation stats
│       │   ├── history/+page.svelte # Historical log
│       │   └── domains/+page.svelte # Which domains manipulate you most
│       └── components/
│           ├── ManipulationScore.svelte
│           ├── TacticBreakdown.svelte
│           ├── AlertTimeline.svelte
│           └── DomainLeaderboard.svelte
│
├── pyproject.toml
├── Makefile
└── README.md
🗄️ Database Schema
SQL

-- Every piece of content analyzed
CREATE TABLE analyses (
    id              TEXT PRIMARY KEY,
    analyzed_at     DATETIME NOT NULL,
    source_type     TEXT NOT NULL,   -- 'webpage' | 'email' | 'notification'
    source_url      TEXT,
    source_domain   TEXT,
    source_title    TEXT,
    content_preview TEXT,            -- First 500 chars
    overall_score   REAL,            -- 0.0 (clean) → 1.0 (maximum manipulation)
    severity        TEXT,            -- 'clean' | 'low' | 'medium' | 'high' | 'critical'
    tactic_count    INTEGER DEFAULT 0,
    processing_ms   INTEGER
);

-- Individual manipulation tactics detected
CREATE TABLE tactics (
    id              TEXT PRIMARY KEY,
    analysis_id     TEXT NOT NULL REFERENCES analyses(id),
    tactic_type     TEXT NOT NULL,
    -- 'urgency' | 'false_scarcity' | 'social_proof' | 'authority' |
    -- 'fear_induction' | 'loss_aversion' | 'dark_pattern' | 'outrage_bait' |
    -- 'reciprocity_trap' | 'gaslighting' | 'clickbait' | 'propaganda'
    confidence      REAL NOT NULL,
    severity        TEXT NOT NULL,
    evidence        TEXT NOT NULL,   -- The specific text/element that triggered it
    explanation     TEXT NOT NULL,   -- Human-readable: "This uses urgency + loss aversion"
    location_hint   TEXT             -- Where in the page: "headline" | "CTA" | "footer"
);

-- Tracking per-domain manipulation history
CREATE TABLE domain_stats (
    domain          TEXT PRIMARY KEY,
    total_analyses  INTEGER DEFAULT 0,
    avg_score       REAL DEFAULT 0.0,
    max_score       REAL DEFAULT 0.0,
    tactic_counts   TEXT,            -- JSON: {tactic_type: count}
    first_seen      DATETIME,
    last_seen       DATETIME,
    trust_rating    TEXT             -- 'trusted' | 'neutral' | 'suspicious' | 'hostile'
);

-- Your daily manipulation exposure report
CREATE TABLE daily_reports (
    date            TEXT PRIMARY KEY,   -- YYYY-MM-DD
    analyses_count  INTEGER DEFAULT 0,
    avg_score       REAL DEFAULT 0.0,
    top_tactic      TEXT,
    highest_domain  TEXT,
    manipulation_minutes INTEGER        -- Estimated minutes exposed to manipulation
);

-- User-defined rules and whitelists
CREATE TABLE user_rules (
    id              TEXT PRIMARY KEY,
    rule_type       TEXT,            -- 'whitelist_domain' | 'suppress_tactic' | 'custom'
    pattern         TEXT NOT NULL,
    created_at      DATETIME NOT NULL
);
🔍 Manipulation Detector Examples
Python

# detectors/linguistic/urgency.py

class UrgencyDetector(BaseDetector):
    """
    Detects manufactured urgency designed to bypass rational decision-making.
    Cialdini's "scarcity principle" weaponized.
    """

    URGENCY_PATTERNS = [
        r'\bact now\b', r'\blimited time\b', r'\bexpires? (today|soon|tonight)\b',
        r'\blast chance\b', r'\bending soon\b', r'\bdon\'t (wait|delay|miss)\b',
        r'\btoday only\b', r'\b(hours?|minutes?) (left|remaining)\b',
        r'\bwhile (supplies|stock|it) (last|lasts)\b', r'\burge[nd]\b',
    ]

    COUNTDOWN_SIGNALS = [
        r'\b\d+:\d{2}:\d{2}\b',           # HH:MM:SS countdown
        r'\b\d+ (hours?|minutes?) left\b',
    ]

    def detect(self, content: AnalyzableContent) -> List[TacticResult]:
        results = []

        # Pattern matching on text
        for pattern in self.URGENCY_PATTERNS:
            matches = re.findall(pattern, content.text, re.IGNORECASE)
            if matches:
                results.append(TacticResult(
                    tactic_type="urgency",
                    confidence=min(0.4 + len(matches) * 0.15, 0.95),
                    evidence=matches[0],
                    explanation=f'Urgency language detected: "{matches[0]}". '
                               f'This is designed to prevent careful consideration '
                               f'by creating time pressure.',
                    severity=self._score_severity(len(matches))
                ))

        return results
Python

# detectors/ux/dark_patterns.py

class DarkPatternDetector(BaseDetector):
    """
    Detects dark UX patterns using DOM analysis from browser extension.
    Receives a simplified DOM structure from the extension.
    """

    def detect(self, dom_data: DOMSnapshot) -> List[TacticResult]:
        results = []

        # 1. Confirm-shaming: opt-out buttons with guilt language
        for element in dom_data.buttons + dom_data.links:
            guilt_patterns = [
                "no thanks", "no, i don't want", "i hate", "i don't want to save",
                "i prefer to pay more", "no, i'll struggle"
            ]
            for pattern in guilt_patterns:
                if pattern in element.text.lower():
                    results.append(TacticResult(
                        tactic_type="dark_pattern",
                        confidence=0.92,
                        evidence=element.text,
                        explanation=f'Confirm-shaming detected: "{element.text}". '
                                   f'This uses guilt to manipulate opt-out decisions.',
                        severity="high"
                    ))

        # 2. Pre-ticked checkboxes (auto-enrolled in marketing)
        for checkbox in dom_data.checkboxes:
            if checkbox.checked and any(kw in checkbox.label.lower()
                for kw in ['newsletter', 'marketing', 'offers', 'updates', 'partner']):
                results.append(TacticResult(
                    tactic_type="dark_pattern",
                    confidence=0.88,
                    evidence=f"Pre-ticked: '{checkbox.label}'",
                    explanation="Pre-ticked marketing checkbox. You are being "
                               "auto-enrolled without active consent.",
                    severity="medium"
                ))

        # 3. Fake countdown timers (JavaScript-driven)
        for timer in dom_data.countdown_elements:
            if timer.resets_on_reload or timer.always_shows_same_value:
                results.append(TacticResult(
                    tactic_type="dark_pattern",
                    confidence=0.91,
                    evidence="Countdown timer that resets",
                    explanation="This countdown timer is fake — it resets when "
                               "the page reloads. The scarcity is manufactured.",
                    severity="high"
                ))

        return results
🖥️ Browser Extension Overlay UI
text

┌──────────────────────────────────────────────────────────────┐
│                EGOSHIELD ALERT                       [×]     │
│  ────────────────────────────────────────────────────────    │
│  🛡️ Manipulation Score: 7.8/10  [CRITICAL]                   │
│                                                              │
│  Detected on: checkout.bigretailer.com                       │
│                                                              │
│  TACTICS FOUND:                                              │
│  🔴 Fake Countdown Timer         "03:47:22 left!"            │
│     → This timer resets. The scarcity is manufactured.      │
│                                                              │
│  🔴 Confirm-Shaming              "No thanks, I hate saving"  │
│     → Guilt-trip opt-out. Designed to feel shameful.        │
│                                                              │
│  🟠 False Social Proof           "847 people viewing now"    │
│     → Unverifiable claim. Classic urgency manufacture.      │
│                                                              │
│  🟠 Loss Aversion Framing        "Don't miss out on 40% off" │
│     → Framed as a loss, not a gain. Triggers loss aversion.  │
│                                                              │
│  🟡 Pre-ticked Upsell            [✓] Add Premium ($4.99/mo)  │
│     → You were auto-enrolled. Uncheck to opt out.           │
│                                                              │
│  [View Full Report]  [Always Trust This Domain]  [Dismiss]  │
└──────────────────────────────────────────────────────────────┘
📊 Dashboard: Your Weekly Manipulation Report
text

┌─────────────────────── EgoShield Dashboard ──────────────────┐
│                                                              │
│  WEEK OF JAN 6-12, 2025                                      │
│                                                              │
│  You were exposed to manipulation on 94 pages this week.     │
│                                                              │
│  TOP MANIPULATION TACTICS USED ON YOU:                       │
│  ─────────────────────────────────────                       │
│  1. Urgency Language          ████████████████░  67 times   │
│  2. False Social Proof        ████████████░░░░░  52 times   │
│  3. Dark Patterns (UX)        █████████░░░░░░░░  38 times   │
│  4. Fear Induction            ██████░░░░░░░░░░░  24 times   │
│  5. Outrage Baiting           █████░░░░░░░░░░░░  19 times   │
│                                                              │
│  WORST OFFENDERS:                                            │
│  ─────────────────                                           │
│  🏴‍☠️ amazon.com           Score: 8.1/10  Mostly: Dark Patterns│
│  🏴‍☠️ dailymail.co.uk      Score: 9.2/10  Mostly: Outrage bait │
│  🏴‍☠️ booking.com          Score: 8.7/10  Mostly: False scarcity│
│                                                              │
│  CLEAN SITES THIS WEEK:                                      │
│  ✅ wikipedia.org   ✅ github.com   ✅ arxiv.org             │
└──────────────────────────────────────────────────────────────┘
📦 Tech Stack Summary
Layer	Technology
Backend	Python 3.11 + FastAPI
NLP / Pattern Detection	spaCy + custom rule engines
LLM Arbiter	Ollama (mistral / llama3)
Database	SQLite
Browser Extension	TypeScript + MV3
DOM Analysis	Sent as structured JSON from extension
Frontend	Svelte + TailwindCSS
Notification	beeep (desktop)
🚀 Build Phases
text

Phase 1 — Linguistic Detectors (3 weeks)
├── Urgency, scarcity, social proof detectors
├── Loss aversion + fear induction
├── Clickbait headline scorer
└── Basic scoring engine

Phase 2 — Browser Extension (2 weeks)
├── Page content extraction
├── DOM snapshot for dark pattern detection
├── Overlay + highlight UI
└── Extension popup

Phase 3 — Dark UX Pattern Detector (2 weeks)
├── Confirm-shaming detector
├── Pre-ticked checkbox detection
├── Countdown timer authenticity check
└── Misdirection + visual hierarchy analysis

Phase 4 — LLM Arbiter + Email (2 weeks)
├── Ollama integration for final-pass LLM analysis
├── IMAP email manipulation scanner
└── Explanation quality tuning

Phase 5 — Dashboard + Reports (2 weeks)
├── Svelte dashboard
├── Weekly manipulation report
├── Domain leaderboard
└── Export + data portability
