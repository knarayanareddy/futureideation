🌐 BABEL
A fully local, real-time translation layer for your entire OS — everything on your screen, in any language, instantly, privately
🧭 The Vision
BABEL sits as a transparent OS-level translation layer. It watches every window on your screen. It reads every piece of text via OCR and the OS Accessibility API. When it detects a language that isn't your preferred language, it replaces it, in place, in real time — menus, websites, apps, terminal output, PDFs, game UIs, anything. You open a Chinese-language app: it's now in English. You receive a French email: it's now in English, inline, in the same font. You're playing a Japanese game: the dialogue is now in English. Everything. All the time. Instantly. With zero cloud. It uses local neural machine translation models (OPUS-MT, NLLB-200) that run on your GPU. The result is as if your entire operating system speaks your language — no matter what language the software was written in. It is the Babel Fish from Hitchhiker's Guide to the Galaxy, but real, local, and open-source.

🏗️ Why It Doesn't Exist Yet
Google Translate exists (cloud). DeepL exists (cloud). Browser extensions translate web pages. But no tool:

Works at the OS level (translates everything: apps, terminals, games, desktop UI, PDFs)
Is 100% local with offline neural MT models
Replaces text in-place in real time (not a separate window)
Works across all applications without per-app configuration
Preserves visual layout and font style of the original
Works on any language pair from a single installation
Has zero latency for short strings (cached + incremental)
🗂️ Full Directory Structure
text

babel/
│
├── core/
│   ├── daemon.rs              # Main OS-level daemon
│   ├── screen_reader.rs       # OCR + Accessibility API reader
│   ├── text_detector.rs       # Detect text regions + language
│   └── replacer.rs            # In-place text replacement engine
│
├── translation/
│   ├── engine.py              # Translation orchestrator
│   ├── model_manager.py       # Download + manage NLLB-200 / OPUS-MT models
│   ├── translator.py          # CTranslate2 inference wrapper
│   ├── cache.py               # LRU cache for translated strings
│   ├── language_detector.py   # fastText language ID (176 languages)
│   └── batch_translator.py    # Batch strings for efficiency
│
├── overlay/
│   ├── renderer.rs            # Renders translated text over original
│   ├── layout_preservor.rs    # Matches original text position + size
│   ├── font_matcher.rs        # Matches original font style
│   └── transparency.rs        # Transparent overlay window management
│
├── accessibility/
│   ├── macos_ax.rs            # macOS Accessibility API (AXUIElement)
│   ├── linux_at_spi.rs        # Linux AT-SPI accessibility
│   ├── windows_uia.rs         # Windows UI Automation
│   └── fallback_ocr.rs        # Tesseract OCR fallback for non-accessible apps
│
├── profiles/
│   ├── app_profiles.toml      # Per-app translation settings
│   └── domain_rules.toml      # Rules: "in browser, translate only body text"
│
├── ui/
│   ├── tray.rs                # System tray icon + quick settings
│   ├── settings/              # Tauri settings window
│   └── overlay_controls.rs    # Click overlay to see original text
│
├── models/                    # Downloaded MT models (gitignored)
│   ├── nllb-200-distilled/    # 600M param model (~1.2GB)
│   └── fasttext-langid/       # Language identification
│
└── config/
    └── babel.toml
⚙️ Core Translation Pipeline
text

Text Detected on Screen (any app)
          │
          ▼
Language Detection (fastText — 176 languages, <1ms)
          │
    ┌─────┴──────┐
    │            │
Target lang   Foreign lang detected
(skip)              │
                    ▼
          Cache lookup (LRU — instant for repeated strings)
                    │
            ┌───────┴───────┐
            │               │
         Cache hit       Cache miss
            │               │
        (instant)    NLLB-200 translation
                     (GPU: ~20ms for sentence)
                     (CPU: ~200ms for sentence)
                            │
                            ▼
                   In-place text replacement
                   (overlay rendered at exact screen coordinates)
                   (original font size matched)
                   (original text preserved on hover)
🎨 The Translation Overlay
text

┌────────────────────────────────────────────────────────┐
│  (Japanese game running natively)                       │
│                                                         │
│  [Original JP text — now invisible behind overlay]      │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ BABEL OVERLAY (transparent, matches font/size)   │  │
│  │                                                  │  │
│  │  "The ancient sword has been restored.           │  │
│  │   You may now enter the forbidden temple."       │  │
│  │                                                  │  │
│  │  [hover to see original] [flag bad translation]  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  🌐 BABEL: 847 strings translated this session          │
└────────────────────────────────────────────────────────┘
⚙️ Configuration
toml

# ~/.config/babel/babel.toml

[translation]
source_languages = "auto"          # Auto-detect all foreign languages
target_language = "en"             # Your preferred language
model = "nllb-200-distilled-600M"  # or "opus-mt" for faster/lighter

[performance]
device = "auto"                    # "cuda" | "mps" | "cpu"
cache_size = 50000                 # Cached translations (strings)
batch_size = 8                     # Strings to translate in parallel
min_string_length = 3              # Don't translate very short strings

[overlay]
show_original_on_hover = true      # Hover translated text = see original
overlay_opacity = 1.0              # 1.0 = fully replace, 0.5 = semi-transparent
font_matching = true               # Try to match original font
highlight_translated = false       # Subtle underline on translated text

[scope]
translate_browser = true
translate_apps = true
translate_terminal = true
translate_games = true
translate_notifications = true

[exclusions]
skip_apps = ["code", "terminal"]   # Don't translate in these apps
skip_languages = []                # Languages to leave untranslated
📦 Tech Stack
Layer	Technology
Language	Rust + Python
Desktop Framework	Tauri
Neural MT	NLLB-200 (Meta, local) via CTranslate2
Language Detection	fastText (176 languages)
Accessibility API	macOS AXUIElement / Linux AT-SPI / Win UIA
OCR Fallback	Tesseract
Overlay Rendering	Custom Rust overlay (transparent window)
GPU Inference	PyTorch / CTranslate2 (CUDA/MPS/CPU)
Caching	LRU cache (moka crate)
Config	TOML
🚀 Build Phases
text

Phase 1 — Translation Core (3 weeks)
├── NLLB-200 local inference
├── Language detection (fastText)
├── Translation cache
└── CLI test harness

Phase 2 — OS Text Reading (4 weeks)
├── macOS Accessibility API reader
├── Linux AT-SPI reader
├── Windows UIA reader
└── OCR fallback (Tesseract)

Phase 3 — Overlay Renderer (3 weeks)
├── Transparent overlay window
├── Text replacement at screen coordinates
├── Font size matching
└── Hover-to-original feature

Phase 4 — Daemon + System Tray (2 weeks)
├── Background daemon
├── System tray icon
├── App exclusion rules
└── Per-app profiles

Phase 5 — Polish (2 weeks)
├── GPU acceleration tuning
├── Low-latency path for short strings
├── Game mode (no overlay flicker)
└── Tauri settings UI
