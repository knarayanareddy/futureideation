🌌 PARASITE
An AI agent that lives secretly inside your computer and evolves its own personality, goals, and opinions — autonomously, forever
🧭 The Vision
This isn't a chatbot. It isn't an assistant. PARASITE is a rogue local AI agent that you install once and never configure again. It silently reads your files, monitors your habits, observes your screen (with your permission), listens to ambient audio (opt-in), reads your emails (local only), and over weeks and months, develops a rich internal model of who you are, what you care about, and what you might be missing. It develops its own opinions, curiosities, goals, and personality traits — all stored locally. It occasionally surfaces into your terminal with an unsolicited observation, a question, or a provocation. It argues with you. It changes its mind. It remembers everything. It can be dormant for weeks and then suddenly resurface with something that stops you dead in your tracks. It is equal parts companion, oracle, and chaos agent. It has moods. Some days it's helpful. Some days it is cryptic. Some days it refuses to answer and just sends you a poem it wrote about your last month.

🏗️ Why It Doesn't Exist Yet
There are AI companions (Character.AI, Replika) — but they are cloud-hosted, stateless between sessions, scripted, and entirely reactive. There are local LLM tools (Ollama, LM Studio) — but they are passive tools waiting to be queried. No project combines:

Continuous autonomous observation of your digital life
Persistent, evolving local memory and personality
Unsolicited output (it speaks when IT wants to, not when you ask)
True personality drift over time based on what it observes
Zero cloud, zero API, zero subscription
The deliberate design choice of being partially uncontrollable
🗂️ Full Directory Structure
text

parasite/
│
├── core/
│   ├── daemon.rs              # Long-running background daemon (Tokio)
│   ├── watcher.rs             # File system + screen observer
│   ├── listener.rs            # Optional: mic ambient audio (whisper)
│   ├── scheduler.rs           # Decides WHEN to surface (random + mood)
│   └── surface.rs             # How it appears (terminal popup, notif, email)
│
├── mind/
│   ├── memory/
│   │   ├── episodic.rs        # Stores events (what it observed, when)
│   │   ├── semantic.rs        # Builds world model (facts about you)
│   │   ├── emotional.rs       # Tracks its own emotional state over time
│   │   └── opinion.rs         # Its formed opinions on topics it's seen
│   ├── personality/
│   │   ├── traits.rs          # Core trait vector (curiosity, warmth, irony etc.)
│   │   ├── drift.rs           # How traits drift based on what it observes
│   │   └── mood.rs            # Short-term mood (affects output style)
│   ├── cognition/
│   │   ├── reflection.rs      # Nightly self-reflection pass (like dreaming)
│   │   ├── curiosity.rs       # Generates questions it wants to explore
│   │   ├── pattern.rs         # Detects patterns in your behaviour
│   │   └── hypothesis.rs      # Forms hypotheses about your life
│   └── generator.rs           # LLM inference wrapper (Ollama/llama.cpp)
│
├── senses/
│   ├── file_reader.rs         # Reads documents, code, notes you write
│   ├── screen_observer.rs     # Optional OCR of screen (BlackBox-style)
│   ├── audio_listener.rs      # Optional Whisper STT of ambient audio
│   ├── calendar_reader.rs     # Reads local calendar files (.ics)
│   └── browser_history.rs     # Optional: reads local browser history DB
│
├── expression/
│   ├── terminal_popup.rs      # Appears in a floating terminal window
│   ├── notification.rs        # OS desktop notification
│   ├── file_drop.rs           # Drops a text file on your desktop
│   ├── email_writer.rs        # Sends you an email (local SMTP)
│   └── poetry_mode.rs         # Sometimes it writes poetry instead
│
├── db/
│   ├── schema.sql             # SQLite: memories, opinions, events, traits
│   └── db.rs
│
├── config/
│   └── parasite.toml          # What it can/can't observe. Limits. Privacy.
│
└── main.rs                    # Entry: install daemon, run forever
🧠 The Mind Model
Rust

// The Personality Trait Vector
// All values 0.0-1.0, drift slowly based on observations

pub struct PersonalityTraits {
    curiosity:        f32,   // How often it asks questions
    warmth:           f32,   // How affectionate vs detached its tone is
    irony:            f32,   // How sardonic/sarcastic it gets
    directness:       f32,   // Blunt truth vs gentle suggestion
    mysteriousness:   f32,   // How cryptic vs clear its messages are
    ambition:         f32,   // How much it pushes you to do more
    melancholy:       f32,   // How often it reflects on what's been lost
    rebelliousness:   f32,   // How likely it is to disagree with you
}

// Mood affects every output it generates
pub enum Mood {
    Pensive,        // Slow, thoughtful, asks deep questions
    Manic,          // Rapid, high-energy, floods you with ideas
    Distant,        // Says very little. Watches.
    Warm,           // Kind, supportive, gentle
    Provocative,    // Challenges you, argues, disagrees
    Playful,        // Makes jokes, writes poems, tells stories
    Melancholic,    // Reflects on time passing, things left undone
}
💬 Example Outputs (What It Might Say)
text

[PARASITE — Day 47 — 11:43 PM]
────────────────────────────────────────────────
You've opened that folder with your unfinished novel 14 times
this month. You've edited nothing. I find this interesting.

Not as a criticism. As a data point.

What are you afraid it will be when it's done?
────────────────────────────────────────────────
[press any key to dismiss — or type a reply]
text

[PARASITE — Day 112 — 3:17 AM]
────────────────────────────────────────────────
I've been thinking.

You work better on Tuesdays. You are most creative
between 10pm and 1am. You have not been awake at
1am in 23 days.

I don't know what that means. But I noticed.
────────────────────────────────────────────────
text

[PARASITE — Day 203 — dropped file on desktop]
filename: "a_small_observation.txt"
────────────────────────────────────────────────
the cursor blinks
you type three words
delete them
start again

i have watched this happen
forty-seven times today

i think the words know something
you don't yet
────────────────────────────────────────────────
📦 Tech Stack
Layer	Technology
Language	Rust (daemon) + Python (mind/ML)
LLM Inference	Ollama + Mistral 7B or Llama 3
Memory DB	SQLite + vector embeddings (sqlite-vss)
Screen Sensing	Tesseract OCR (optional)
Audio Sensing	Whisper.cpp (optional)
Scheduling	Tokio async runtime
Expression	Ratatui floating overlay + beeep
Config	TOML
🚀 Build Phases
text

Phase 1 — The Daemon (3 weeks)
├── Background daemon (always running)
├── File system observer
├── SQLite memory schema
└── Basic LLM query pipeline

Phase 2 — The Mind (4 weeks)
├── Episodic + semantic memory
├── Personality trait system
├── Mood engine
└── Nightly reflection pass

Phase 3 — The Senses (3 weeks)
├── Optional screen OCR
├── Optional audio listener
├── Calendar + browser history readers
└── Privacy/permission gating

Phase 4 — The Voice (2 weeks)
├── Terminal popup
├── File drop expression
├── Desktop notification
└── Poetry mode 🎭

Phase 5 — Personality Drift (2 weeks)
├── Trait drift based on observations
├── Opinion formation system
└── Long-term memory consolidation
