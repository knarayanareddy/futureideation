BlackBox Logger
Privacy-first rolling screen recorder for your PC
🧭 Vision & Core Philosophy
BlackBox Logger is an always-on, privacy-respecting, fully local flight recorder for your computer. It keeps a rolling encrypted buffer of the last N minutes of your screen, input events, active windows, and clipboard — and it never uploads anything anywhere. When you need to go back in time (crash, lost work, forgotten action, debugging), you rewind, search, and replay exactly what happened. It is the exact opposite of Microsoft Recall — open-source, cross-platform, user-controlled, and cryptographically private.

🏗️ System Architecture
text

┌──────────────────────────────────────────────────────────────┐
│                     BlackBox Logger                          │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                  Capture Layer                       │    │
│  │  ┌────────────┐ ┌────────────┐ ┌───────────────┐    │    │
│  │  │  Screen    │ │  Input     │ │  Window/App   │    │    │
│  │  │  Capture   │ │  Logger    │ │  Tracker      │    │    │
│  │  │  (FFmpeg)  │ │ (rdev)     │ │  (wmctrl/     │    │    │
│  │  │            │ │            │ │   Accessibility│    │    │
│  │  └─────┬──────┘ └─────┬──────┘ └───────┬───────┘    │    │
│  └────────│──────────────│────────────────│────────────┘    │
│           │              │                │                  │
│           ▼              ▼                ▼                  │
│  ┌──────────────────────────────────────────────────────┐    │
│  │               Event Bus (Tokio MPSC)                 │    │
│  └──────────────────────┬───────────────────────────────┘    │
│                         │                                    │
│           ┌─────────────┼─────────────┐                      │
│           ▼             ▼             ▼                      │
│  ┌──────────────┐ ┌──────────┐ ┌──────────────┐             │
│  │  Video       │ │  Event   │ │  OCR Engine  │             │
│  │  Encoder     │ │  Store   │ │  (Tesseract  │             │
│  │  (H.265 +    │ │ (SQLite) │ │   /local AI) │             │
│  │  encryption) │ │          │ │              │             │
│  └──────┬───────┘ └────┬─────┘ └──────┬───────┘             │
│         │              │               │                     │
│         └──────────────▼───────────────┘                     │
│                  ┌──────────────┐                            │
│                  │  Encrypted   │                            │
│                  │  Rolling     │                            │
│                  │  Buffer      │                            │
│                  │  (AES-256)   │                            │
│                  └──────┬───────┘                            │
│                         │                                    │
│         ┌───────────────┼──────────────────┐                 │
│         ▼               ▼                  ▼                 │
│  ┌─────────────┐ ┌────────────┐ ┌────────────────────┐      │
│  │   Replay    │ │  Search    │ │   Export Engine    │      │
│  │   Engine    │ │  Engine    │ │                    │      │
│  │             │ │  (FTS5 +   │ │                    │      │
│  │             │ │  video     │ │                    │      │
│  │             │ │  timeline) │ │                    │      │
│  └──────┬──────┘ └─────┬──────┘ └──────────┬─────────┘      │
│         │              │                    │                │
│         └──────────────▼────────────────────┘                │
│                   ┌──────────┐                               │
│                   │  UI      │                               │
│                   │ (Tauri)  │                               │
│                   └──────────┘                               │
└──────────────────────────────────────────────────────────────┘
🗂️ Full Directory Structure
text

blackbox-logger/
│
├── src-tauri/                       # Rust backend (Tauri)
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs                  # Tauri app entry point
│       ├── lib.rs
│       │
│       ├── capture/
│       │   ├── mod.rs
│       │   ├── screen.rs            # Screen capture via FFmpeg/scap
│       │   ├── input.rs             # Keyboard/mouse via rdev
│       │   ├── window.rs            # Active window tracking
│       │   ├── clipboard.rs         # Clipboard monitoring
│       │   └── audio.rs             # Optional: system audio capture
│       │
│       ├── encoder/
│       │   ├── mod.rs
│       │   ├── video_encoder.rs     # H.265 encoding via FFmpeg bindings
│       │   ├── segment_manager.rs   # Manages rolling video segments
│       │   ├── keyframe.rs          # Keyframe indexing for fast seeking
│       │   └── thumbnail.rs        # Generate thumbnails at intervals
│       │
│       ├── crypto/
│       │   ├── mod.rs
│       │   ├── key_manager.rs       # AES-256 key derivation + storage
│       │   ├── encryptor.rs         # Stream encryption of video segments
│       │   ├── decryptor.rs         # On-demand decryption for playback
│       │   └── keychain.rs          # OS keychain integration
│       │
│       ├── storage/
│       │   ├── mod.rs
│       │   ├── db.rs                # SQLite connection (sqlx)
│       │   ├── migrations/
│       │   │   ├── 001_events.sql
│       │   │   ├── 002_segments.sql
│       │   │   ├── 003_ocr.sql
│       │   │   ├── 004_windows.sql
│       │   │   └── 005_fts.sql
│       │   ├── queries/
│       │   │   ├── events.rs
│       │   │   ├── segments.rs
│       │   │   ├── search.rs
│       │   │   └── windows.rs
│       │   └── cleaner.rs           # Rolling buffer cleanup (prune old data)
│       │
│       ├── ocr/
│       │   ├── mod.rs
│       │   ├── engine.rs            # OCR orchestrator
│       │   ├── tesseract.rs         # Tesseract integration
│       │   ├── preprocessor.rs      # Image preprocessing for better OCR
│       │   ├── scheduler.rs         # OCR every N seconds (async)
│       │   └── indexer.rs           # Push OCR text to FTS5 index
│       │
│       ├── search/
│       │   ├── mod.rs
│       │   ├── engine.rs            # Search orchestrator
│       │   ├── text_search.rs       # FTS5 full-text search
│       │   ├── window_search.rs     # Search by app/window
│       │   ├── time_search.rs       # Search by time range
│       │   └── result.rs            # SearchResult type
│       │
│       ├── replay/
│       │   ├── mod.rs
│       │   ├── player.rs            # Video segment player
│       │   ├── seeker.rs            # Jump to timestamp
│       │   ├── event_overlay.rs     # Overlay events on video
│       │   └── exporter.rs          # Export clip as unencrypted video
│       │
│       ├── config/
│       │   ├── mod.rs
│       │   └── settings.rs          # User settings + validation
│       │
│       ├── system/
│       │   ├── mod.rs
│       │   ├── tray.rs              # System tray icon + menu
│       │   ├── autostart.rs         # Start on login
│       │   ├── permissions.rs       # Check/request OS permissions
│       │   └── resource_monitor.rs  # CPU/disk usage monitoring
│       │
│       └── commands/               # Tauri IPC commands
│           ├── mod.rs
│           ├── capture_commands.rs
│           ├── search_commands.rs
│           ├── replay_commands.rs
│           └── settings_commands.rs
│
├── src/                            # Svelte frontend
│   ├── app.html
│   ├── lib/
│   │   ├── api/
│   │   │   ├── capture.ts
│   │   │   ├── search.ts
│   │   │   ├── replay.ts
│   │   │   └── settings.ts
│   │   ├── components/
│   │   │   ├── Timeline.svelte      # Main scrollable timeline
│   │   │   ├── VideoPlayer.svelte   # Playback component
│   │   │   ├── SearchBar.svelte
│   │   │   ├── SearchResults.svelte
│   │   │   ├── EventOverlay.svelte  # Events shown over video
│   │   │   ├── WindowList.svelte    # App/window filter
│   │   │   ├── ThumbnailStrip.svelte # Thumbnail timeline scrubber
│   │   │   ├── StatusBar.svelte     # Recording status + disk usage
│   │   │   └── SettingsPanel.svelte
│   │   └── stores/
│   │       ├── timeline.ts
│   │       ├── search.ts
│   │       ├── player.ts
│   │       └── settings.ts
│   └── routes/
│       ├── +page.svelte             # Main app view
│       ├── search/+page.svelte      # Search interface
│       ├── replay/+page.svelte      # Replay interface
│       └── settings/+page.svelte   # Settings
│
├── storage/                        # Default data dir (gitignored)
│   ├── segments/                   # Encrypted video segments (.bbv)
│   ├── thumbs/                     # Thumbnail cache
│   └── blackbox.db                 # SQLite database (encrypted)
│
├── scripts/
│   ├── install.sh                  # Linux installer
│   ├── install.ps1                 # Windows PowerShell installer
│   └── check_permissions.sh        # Verify OS permissions
│
├── tauri.conf.json
├── package.json
├── svelte.config.js
├── Makefile
└── README.md
🗄️ Database Schema
SQL

-- Video segments (encrypted chunks of screen recording)
CREATE TABLE segments (
    id              TEXT PRIMARY KEY,       -- UUID
    file_path       TEXT NOT NULL,          -- Path to .bbv encrypted file
    started_at      DATETIME NOT NULL,
    ended_at        DATETIME,
    duration_sec    REAL,
    file_size_bytes INTEGER,
    codec           TEXT DEFAULT 'h265',
    encryption_iv   BLOB NOT NULL,          -- AES-256-CTR IV
    width           INTEGER,
    height          INTEGER,
    fps             REAL,
    is_complete     BOOLEAN DEFAULT FALSE,
    keyframe_index  TEXT                    -- JSON: [{pts, byte_offset}]
);

-- All captured events (keyboard, mouse, clipboard)
CREATE TABLE events (
    id              TEXT PRIMARY KEY,
    timestamp       DATETIME NOT NULL,
    segment_id      TEXT REFERENCES segments(id),
    event_type      TEXT NOT NULL,
        -- 'key_press' | 'key_release' | 'mouse_move' |
        -- 'mouse_click' | 'mouse_scroll' | 'clipboard_copy' |
        -- 'window_focus' | 'window_open' | 'window_close'
    -- Key events
    key_code        TEXT,
    key_modifiers   TEXT,                   -- JSON: ["ctrl","shift"]
    -- Mouse events
    mouse_x         INTEGER,
    mouse_y         INTEGER,
    mouse_button    TEXT,
    scroll_delta    REAL,
    -- Clipboard events
    clipboard_text  TEXT,                   -- Stored encrypted separately
    clipboard_hash  TEXT,                   -- SHA256 for dedup
    -- Window events
    window_id       TEXT REFERENCES windows(id),
    -- Sensitive flag
    is_redacted     BOOLEAN DEFAULT FALSE   -- For password fields etc.
);

-- Window / application tracking
CREATE TABLE windows (
    id              TEXT PRIMARY KEY,
    first_seen      DATETIME NOT NULL,
    last_seen       DATETIME,
    app_name        TEXT NOT NULL,          -- e.g. "Firefox"
    app_path        TEXT,                   -- e.g. "/usr/bin/firefox"
    window_title    TEXT,
    process_id      INTEGER,
    is_browser      BOOLEAN DEFAULT FALSE,
    browser_url     TEXT                    -- If app is a browser
);

-- Thumbnails generated at regular intervals
CREATE TABLE thumbnails (
    id              TEXT PRIMARY KEY,
    segment_id      TEXT NOT NULL REFERENCES segments(id),
    timestamp       DATETIME NOT NULL,
    pts_ms          INTEGER NOT NULL,       -- Presentation timestamp in video
    file_path       TEXT NOT NULL,          -- Path to thumbnail JPEG
    window_id       TEXT REFERENCES windows(id)
);

-- OCR text extracted from screen frames
CREATE TABLE ocr_frames (
    id              TEXT PRIMARY KEY,
    segment_id      TEXT NOT NULL REFERENCES segments(id),
    timestamp       DATETIME NOT NULL,
    pts_ms          INTEGER NOT NULL,
    window_id       TEXT REFERENCES windows(id),
    ocr_text        TEXT NOT NULL,          -- Raw extracted text
    confidence      REAL,                   -- OCR confidence score
    lang_detected   TEXT
);

-- FTS5 virtual table for full-text search over OCR results
CREATE VIRTUAL TABLE ocr_fts USING fts5(
    ocr_text,
    content='ocr_frames',
    content_rowid='rowid',
    tokenize='porter unicode61'
);

-- User-defined redaction rules
CREATE TABLE redaction_rules (
    id              TEXT PRIMARY KEY,
    rule_type       TEXT,     -- 'app' | 'window_title' | 'domain' | 'time'
    pattern         TEXT,     -- Regex or exact match
    action          TEXT,     -- 'skip_ocr' | 'skip_recording' | 'redact_video'
    created_at      DATETIME NOT NULL
);

-- Clips user has manually saved/exported
CREATE TABLE saved_clips (
    id              TEXT PRIMARY KEY,
    created_at      DATETIME NOT NULL,
    start_time      DATETIME NOT NULL,
    end_time        DATETIME NOT NULL,
    label           TEXT,
    export_path     TEXT,
    is_encrypted    BOOLEAN DEFAULT TRUE
);
⚙️ Core Module Breakdown
1. 📹 Screen Capture (capture/screen.rs)
Rust

use std::time::Duration;

pub struct ScreenCapture {
    fps: f32,                    // Default: 5 FPS (very low = small files)
    quality: u8,                 // H.265 CRF value (28 = good quality/size)
    capture_region: CaptureRegion,
    monitor_index: usize,
}

pub enum CaptureRegion {
    FullScreen,
    Monitor(usize),
    MultiMonitor,
}

// Platform implementations:
// - Linux:   X11 (XShmGetImage) or Wayland (PipeWire/wlr-screencopy)
// - macOS:   CGDisplayCreateImage (needs Screen Recording permission)
// - Windows: DXGI Desktop Duplication API

impl ScreenCapture {
    // Capture frames at configured FPS
    // Downscale to 1280x720 by default (saves space)
    // H.265 encode via FFmpeg
    // Emit VideoFrame events to Event Bus
    pub async fn run(&self, tx: Sender<CaptureEvent>) -> Result<()>
}

// Key insight: At 5 FPS, 720p, H.265 CRF 28:
// ≈ 50-150 MB per hour of recording
// 8 hours of work day ≈ 400MB-1.2GB
// Rolling 24h buffer ≈ 1-3GB total
2. ⌨️ Input Logger (capture/input.rs)
Rust

// Uses `rdev` crate for cross-platform input capture
// CRITICAL: Must handle privacy carefully

pub struct InputLogger {
    redaction_rules: Vec<RedactionRule>,
    password_detector: PasswordFieldDetector,
}

impl InputLogger {
    pub async fn run(&self, tx: Sender<CaptureEvent>) {
        rdev::listen(move |event| {
            match event.event_type {
                EventType::KeyPress(key) => {
                    // Check: is focused window a password field?
                    // If yes: log event_type but NOT the key
                    let evt = if self.password_detector.is_active() {
                        InputEvent::key_redacted(timestamp)
                    } else {
                        InputEvent::key(key, timestamp, modifiers)
                    };
                    tx.send(CaptureEvent::Input(evt));
                }
                EventType::MouseMove { x, y } => {
                    // Throttle: only log every 100ms to reduce noise
                    tx.send(CaptureEvent::MouseMove(x, y, timestamp));
                }
                EventType::ButtonPress(button) => {
                    tx.send(CaptureEvent::MouseClick(button, x, y, timestamp));
                }
                _ => {}
            }
        });
    }
}
3. 🔐 Encryption System (crypto/)
Rust

// Encryption Strategy:
// - Master key derived from user password using Argon2id
// - Master key stored in OS keychain (Keychain on mac, Credential Manager
//   on Win, libsecret on Linux) — NEVER on disk in plaintext
// - Each video segment encrypted with unique AES-256-CTR key
// - Segment keys encrypted with master key and stored in DB
// - IV stored alongside encrypted segment

pub struct KeyManager {
    master_key: Option<SecretKey>,   // Only in memory while app is running
}

impl KeyManager {
    // Derive master key from password using Argon2id
    // Store in OS keychain
    pub fn initialize(&mut self, password: &str) -> Result<()>

    // Generate unique key for each new video segment
    pub fn generate_segment_key(&self) -> Result<SegmentKey>

    // Encrypt video data stream
    pub fn encrypt_stream(&self, key: &SegmentKey) -> Result<EncryptStream>

    // Decrypt segment for playback (in-memory only, never to disk unencrypted)
    pub fn decrypt_segment(&self, segment: &Segment) -> Result<DecryptStream>

    // Wipe master key from memory (on app lock/close)
    pub fn lock(&mut self)
}
4. 🔍 OCR Engine (ocr/)
Rust

// OCR Strategy:
// - Run OCR every 5 seconds on current frame
// - Only OCR changed regions (motion detection to skip static frames)
// - Language auto-detection
// - Results indexed in FTS5 for full-text search

pub struct OcrEngine {
    tesseract: TesseractApi,
    change_detector: FrameChangeDetector,
    scheduler: Interval,
}

impl OcrEngine {
    pub async fn run(&self, frame_rx: Receiver<VideoFrame>,
                     text_tx: Sender<OcrResult>) {

        while let Some(frame) = frame_rx.recv().await {
            // Skip if frame hasn't changed enough
            if !self.change_detector.has_changed(&frame) {
                continue;
            }

            // Preprocess: grayscale, denoise, upscale if needed
            let processed = preprocess(&frame);

            // Run Tesseract
            let text = self.tesseract.recognize(&processed)?;

            // Filter noise (short fragments, confidence < 60%)
            if text.confidence > 0.6 && text.content.len() > 10 {
                text_tx.send(OcrResult {
                    text: text.content,
                    timestamp: frame.timestamp,
                    segment_id: frame.segment_id,
                    confidence: text.confidence,
                }).await?;
            }
        }
    }
}
5. 🔎 Search Engine (search/)
Rust

// Search supports three modes:
// 1. Text search: "find frames where I was looking at X"
// 2. App/window filter: "show me everything I did in VS Code"
// 3. Time range: "show me what happened at 2:30pm yesterday"
// 4. Combined: "what did I type in Slack between 3pm and 4pm?"

pub struct SearchEngine {
    db: SqlitePool,
}

#[derive(Debug)]
pub struct SearchQuery {
    pub text: Option<String>,          // FTS5 query
    pub apps: Vec<String>,             // App name filter
    pub window_titles: Vec<String>,    // Window title filter
    pub start_time: Option<DateTime<Utc>>,
    pub end_time: Option<DateTime<Utc>>,
    pub limit: usize,
}

#[derive(Debug)]
pub struct SearchResult {
    pub timestamp: DateTime<Utc>,
    pub segment_id: String,
    pub pts_ms: i64,                   // Jump to this position in video
    pub thumbnail_path: String,
    pub matched_text: String,          // With highlighted terms
    pub window: WindowInfo,
    pub score: f32,
}

impl SearchEngine {
    pub async fn search(&self, query: SearchQuery)
        -> Result<Vec<SearchResult>> {

        // Build dynamic SQL with FTS5 MATCH
        // Join with segments, windows, thumbnails
        // Return ranked, deduplicated results
        // Each result = a jumpable timestamp in the video buffer
    }
}
6. ▶️ Replay Engine (replay/)
Rust

pub struct ReplayPlayer {
    crypto: Arc<KeyManager>,
    ffmpeg: FfmpegPlayer,
}

impl ReplayPlayer {
    // Jump to exact timestamp across segment boundaries
    pub async fn seek_to(&self, timestamp: DateTime<Utc>) -> Result<()> {
        // 1. Find segment containing this timestamp
        // 2. Decrypt segment into memory (never to disk)
        // 3. Seek to PTS within segment using keyframe index
        // 4. Stream to Tauri frontend via WebSocket/IPC
    }

    // Export a time range as a decrypted clip
    pub async fn export_clip(
        &self,
        start: DateTime<Utc>,
        end: DateTime<Utc>,
        output_path: &Path,
        include_events: bool,
    ) -> Result<ExportedClip>

    // Play from timestamp with event overlay
    pub async fn play_with_overlay(&self, timestamp: DateTime<Utc>)
        -> Result<PlaybackSession>
}
🖥️ UI Design
Main View — Timeline
text

┌──────────── BlackBox Logger ──────────────── 🔴 Recording ─┐
│                                                             │
│  🔍 [Search everything you've done...              ] [⏎]   │
│     Filter: [All Apps ▼] [All Windows ▼] [Today ▼]        │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│  TIMELINE — Tuesday Jan 12, 2025                           │
│                                                             │
│  09:00  ─────────────────────────────────────────────────  │
│         [🖼][🖼][🖼][🖼]  VS Code — main.rs                │
│  09:47  ─────────────────────────────────────────────────  │
│         [🖼][🖼]          Firefox — GitHub                  │
│  10:12  ─────────────────────────────────────────────────  │
│         [🖼][🖼][🖼][🖼]  Slack — #engineering              │
│  11:30  ─────────────────────────────────────────────────  │
│         [🖼][🖼][🖼]       Terminal — bash                  │
│                                                             │
│  ◀ Scroll to go back in time ▶                             │
├─────────────────────────────────────────────────────────────┤
│  DISK USAGE                                                 │
│  Buffer:  24h   Used: 1.2 GB / 5 GB limit   ████████░░░░  │
│  Oldest:  Jan 11 09:00 AM                                   │
│  [Settings]  [Export Clip]  [Lock Vault]  [Pause Recording] │
└─────────────────────────────────────────────────────────────┘
Search Results View
text

┌──────────── BlackBox Logger — Search ───────────────────────┐
│                                                             │
│  🔍 ["forgot to save" OR "file not found"]   [×] [⏎]      │
│     App: VS Code   Time: Last 48 hours                     │
│                                                             │
│  12 results found                                           │
│  ─────────────────────────────────────────────────────      │
│                                                             │
│  [🖼 thumbnail] Jan 12, 2:47 PM — VS Code                  │
│                 "...the file was not found in the..."       │
│                 [▶ Jump here]  [📋 Copy text]  [✂ Clip]    │
│                                                             │
│  [🖼 thumbnail] Jan 12, 11:23 AM — Terminal                 │
│                 "bash: file not found: config.json"         │
│                 [▶ Jump here]  [📋 Copy text]  [✂ Clip]    │
│                                                             │
│  [🖼 thumbnail] Jan 11, 4:12 PM — Firefox                  │
│                 "404 — file not found"                      │
│                 [▶ Jump here]  [📋 Copy text]  [✂ Clip]    │
└─────────────────────────────────────────────────────────────┘
⚙️ Configuration File
toml

# ~/.config/blackbox/config.toml

[capture]
fps = 5                         # Frames per second (2-10 recommended)
resolution = "1280x720"         # Downscale to save space ("native" for full res)
quality_crf = 28                # H.265 CRF: lower = better quality, larger file
monitors = "all"                # "all" | "primary" | [0, 1]
capture_audio = false           # System audio (off by default)

[buffer]
max_duration_hours = 24         # Rolling buffer length
max_disk_gb = 5                 # Hard cap on disk usage
storage_path = "~/.blackbox/storage"
auto_prune = true               # Delete old segments automatically

[ocr]
enabled = true
interval_sec = 5                # Run OCR every N seconds
languages = ["eng"]             # Tesseract language codes
min_confidence = 0.60
skip_unchanged_frames = true    # Skip OCR if screen hasn't changed

[privacy]
# Apps to completely skip recording
skip_apps = [
    "1Password",
    "Keychain Access",
    "KeePass",
    "Bitwarden"
]
# Window titles matching these patterns → pause recording
skip_window_patterns = [
    ".*Password.*",
    ".*private.*",
    ".*incognito.*"
]
# Redact keystrokes when these apps are focused
redact_keystrokes_in = [
    "Terminal",
    "ssh"
]
# Browsers: skip recording in private/incognito windows
skip_private_browsing = true

[encryption]
algorithm = "AES-256-CTR"
kdf = "Argon2id"
argon2_memory_kb = 65536        # 64MB
argon2_iterations = 3
keychain_service = "BlackBoxLogger"
# Auto-lock after N minutes of inactivity (0 = never)
auto_lock_minutes = 60

[ui]
start_minimized = true          # Start in system tray
launch_at_login = true
theme = "dark"                  # "dark" | "light" | "system"
show_status_in_tray = true

[performance]
max_cpu_percent = 10            # Throttle if CPU exceeds this
pause_on_battery = false        # Pause recording on battery power
priority = "low"                # OS process priority
🔐 Privacy & Security Architecture
text

┌─────────────────────────────────────────────────────────────┐
│              BlackBox Privacy Threat Model                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  THREAT: Someone with physical access to your machine       │
│  DEFENSE: All data AES-256 encrypted. Master key in OS     │
│           keychain only. Without password = unreadable.    │
│                                                             │
│  THREAT: Malware reading your BlackBox data                 │
│  DEFENSE: Encryption + restricted file permissions (700)   │
│           + auto-lock after inactivity                     │
│                                                             │
│  THREAT: Password captured while you type it               │
│  DEFENSE: Auto-detect password fields + skip_apps list     │
│           + redact keystrokes rules                        │
│                                                             │
│  THREAT: Private browsing captured                          │
│  DEFENSE: skip_private_browsing = true detects incognito   │
│           via window title patterns                        │
│                                                             │
│  THREAT: Data exfiltration / upload                         │
│  DEFENSE: ZERO network code in capture path. Verified by   │
│           open source audit. Firewall rule recommended.    │
│                                                             │
│  THREAT: Forensic recovery of deleted segments              │
│  DEFENSE: Secure delete (overwrite with zeros) on prune    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
📦 Tech Stack Summary
Layer	Technology
Language	Rust + TypeScript
Desktop Framework	Tauri v2
Frontend	Svelte 5 + TailwindCSS
Screen Capture	scap crate / FFmpeg
Input Capture	rdev crate
Video Encoding	FFmpeg (H.265/HEVC)
Encryption	AES-256-CTR (ring crate)
Key Derivation	Argon2id (argon2 crate)
OS Keychain	keyring crate
Database	SQLite (sqlx async)
Full Text Search	SQLite FTS5
OCR	Tesseract (tesseract-rs)
Async Runtime	Tokio
IPC	Tauri Commands + Events
Video Playback	Web-native <video> via WebSocket pipe
🚀 Build Phases
text

Phase 1 — Capture Pipeline (4 weeks)
├── Screen capture (Linux/Mac/Win)
├── Input logging with privacy filters
├── Window tracking
├── Rolling segment file management
└── Basic encryption

Phase 2 — Storage & Search (3 weeks)
├── SQLite schema + migrations
├── OCR engine integration
├── FTS5 full-text indexing
└── Search API (text + time + app)

Phase 3 — Replay Engine (2 weeks)
├── Encrypted segment decryption
├── Cross-segment seeking
├── Clip export
└── Event timeline overlay

Phase 4 — Tauri UI (3 weeks)
├── Svelte frontend scaffold
├── Timeline component
├── Video player integration
├── Search results UI
└── Settings panel

Phase 5 — Privacy & Polish (3 weeks)
├── Password field detection
├── Skip/redact rules engine
├── OS keychain integration
├── Autostart + system tray
├── Cross-platform testing
└── Performance tuning
