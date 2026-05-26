OracleShell
A terminal that predicts what you're about to type — trained entirely on your own command history
🧭 Vision & Core Philosophy
OracleShell is a drop-in shell wrapper (works with bash, zsh, fish) that builds a deeply personal, local predictive model of how you specifically use the terminal. Not generic autocomplete. Not man-page lookups. Not LLM hallucinations.

It learns your patterns: the sequences of commands you run, the arguments you use, the directories you navigate to, the git workflows you follow, the docker commands you execute in a specific order. Over time it builds a personal command grammar — a probabilistic model of your terminal behavior — and starts predicting your next command before you type it.

But it goes further. OracleShell has a "Context Awareness" layer that reads:

What directory you're in
What git branch you're on
What's in your current docker ps, kubectl get pods, or ls output
What time of day it is
What you've been doing for the last 5 minutes
And it synthesizes all of this into surgical, context-aware predictions that feel almost psychic. It gets genuinely better the more you use it — and it learns continuously in the background with no interruption.

🏗️ System Architecture
text

┌──────────────────────────────────────────────────────────────┐
│                     OracleShell                              │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Shell Integration Layer                 │    │
│  │  (Hooks into bash/zsh/fish via preexec/precmd)       │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             │                               │
│       ┌─────────────────────┼─────────────────────┐         │
│       ▼                     ▼                     ▼         │
│  ┌──────────┐      ┌───────────────┐      ┌──────────────┐  │
│  │ Command  │      │  Context      │      │  Prediction  │  │
│  │ Recorder │      │  Collector    │      │  Requester   │  │
│  │          │      │  (dir, git,   │      │  (async,     │  │
│  │          │      │   env, time)  │      │   < 50ms)    │  │
│  └────┬─────┘      └───────┬───────┘      └──────┬───────┘  │
│       │                    │                     │          │
│       └────────────────────▼─────────────────────┘          │
│                            │                                │
│                            ▼                                │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Oracle Daemon (Rust)                    │    │
│  │                                                      │    │
│  │  ┌─────────────────┐   ┌──────────────────────────┐  │    │
│  │  │  Command Store  │   │  Prediction Engine       │  │    │
│  │  │  (SQLite)       │   │  ┌──────────────────────┐│  │    │
│  │  │                 │   │  │  N-gram Model        ││  │    │
│  │  │                 │   │  │  (fast, local)       ││  │    │
│  │  │                 │   │  ├──────────────────────┤│  │    │
│  │  │                 │   │  │  Markov Chain Model  ││  │    │
│  │  │                 │   │  │  (context-aware)     ││  │    │
│  │  │                 │   │  ├──────────────────────┤│  │    │
│  │  │                 │   │  │  Neural Predictor    ││  │    │
│  │  └─────────────────┘   │  │  (LSTM - optional)  ││  │    │
│  │                        │  └──────────────────────┘│  │    │
│  │                        └──────────────────────────┘  │    │
│  │                                                      │    │
│  │  ┌─────────────────────────────────────────────────┐ │    │
│  │  │            Continuous Trainer                   │ │    │
│  │  │  (Runs in background, updates model as you use  │ │    │
│  │  │   the shell — zero interruption)                │ │    │
│  │  └─────────────────────────────────────────────────┘ │    │
│  └──────────────────────────────────────────────────────┘    │
│                             │                               │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Shell Autocomplete UI                   │    │
│  │  (Inline ghost text + dropdown ranked suggestions)   │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
🗂️ Full Directory Structure
text

oracleshell/
│
├── daemon/                          # Rust daemon
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs                  # Daemon entry point
│       ├── server.rs                # Unix socket server
│       │
│       ├── store/
│       │   ├── mod.rs
│       │   ├── db.rs                # SQLite connection (sqlx)
│       │   ├── command_store.rs     # Store/query commands
│       │   └── context_store.rs     # Store context snapshots
│       │
│       ├── predictor/
│       │   ├── mod.rs
│       │   ├── ngram.rs             # N-gram model (bigram, trigram)
│       │   ├── markov.rs            # Context-aware Markov chain
│       │   ├── neural.rs            # LSTM predictor (optional)
│       │   ├── ensemble.rs          # Ensemble: combine all models
│       │   ├── ranker.rs            # Rank predictions by score + context
│       │   └── command_parser.rs    # Parse commands into tokens
│       │
│       ├── context/
│       │   ├── mod.rs
│       │   ├── collector.rs         # Context collection orchestrator
│       │   ├── directory.rs         # CWD + directory contents
│       │   ├── git.rs               # Git status, branch, recent commits
│       │   ├── docker.rs            # docker ps output
│       │   ├── kubectl.rs           # kubectl get pods
│       │   ├── env.rs               # Environment variables
│       │   └── temporal.rs          # Time of day, day of week
│       │
│       ├── trainer/
│       │   ├── mod.rs
│       │   ├── scheduler.rs         # Background training scheduler
│       │   ├── ngram_trainer.rs     # Train n-gram model from history
│       │   ├── markov_trainer.rs    # Train Markov model
│       │   └── neural_trainer.rs   # Train LSTM (runs nightly)
│       │
│       └── models/
│           ├── mod.rs
│           ├── ngram_model.rs       # Serializable n-gram tables
│           ├── markov_model.rs      # Serializable Markov chain
│           └── prediction.rs        # Prediction result type
│
├── shell/                           # Shell integration scripts
│   ├── oracleshell.bash             # bash integration
│   ├── oracleshell.zsh              # zsh integration (uses zle)
│   ├── oracleshell.fish             # fish integration
│   └── client.py                   # Python client to query daemon
│       # (fast startup via cached connection)
│
├── cli/                             # oracleshell CLI tool
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs
│       └── commands/
│           ├── stats.rs             # Show prediction accuracy stats
│           ├── history.rs           # Browse your command history
│           ├── train.rs             # Force retrain models
│           └── config.rs            # Configuration
│
├── config/
│   └── default.toml
│
├── tests/
├── Makefile
└── README.md
🗄️ Database Schema
SQL

-- Every command ever executed
CREATE TABLE commands (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    command         TEXT NOT NULL,
    -- Tokenized command components
    program         TEXT NOT NULL,   -- e.g. "git", "docker", "kubectl"
    subcommand      TEXT,            -- e.g. "commit", "push", "run"
    args_json       TEXT,            -- JSON array of arguments
    flags_json      TEXT,            -- JSON array of flags
    -- Context at time of execution
    cwd             TEXT,
    git_branch      TEXT,
    git_repo        TEXT,
    hostname        TEXT,
    exit_code       INTEGER,
    duration_ms     INTEGER,
    executed_at     DATETIME NOT NULL,
    hour_of_day     INTEGER,         -- 0-23 (for temporal patterns)
    day_of_week     INTEGER          -- 0-6
);

-- Command sequences (for n-gram and Markov models)
CREATE TABLE command_sequences (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id      TEXT NOT NULL,   -- Group commands by terminal session
    command_id      INTEGER NOT NULL REFERENCES commands(id),
    sequence_index  INTEGER NOT NULL -- Position within session
);

-- Trained N-gram model data
CREATE TABLE ngram_model (
    ngram_hash      TEXT PRIMARY KEY, -- Hash of n-gram key
    context_json    TEXT NOT NULL,    -- JSON: ["git", "add"] (previous commands)
    next_command    TEXT NOT NULL,    -- Predicted next command
    count           INTEGER DEFAULT 1,
    probability     REAL,
    last_seen       DATETIME
);

-- Prediction accuracy tracking
CREATE TABLE predictions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    predicted_at    DATETIME NOT NULL,
    context_json    TEXT NOT NULL,
    prediction_rank INTEGER,         -- Which suggestion was accepted (1-5)
    predicted       TEXT NOT NULL,
    actual          TEXT NOT NULL,
    was_correct     BOOLEAN NOT NULL,
    model_used      TEXT             -- 'ngram' | 'markov' | 'neural' | 'ensemble'
);

-- Context snapshots
CREATE TABLE context_snapshots (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    command_id      INTEGER NOT NULL REFERENCES commands(id),
    context_json    TEXT NOT NULL    -- Full context at prediction time
);
⚡ Prediction Engine Core
Rust

// predictor/ensemble.rs

pub struct EnsemblePredictor {
    ngram: NgramModel,
    markov: MarkovModel,
    neural: Option<NeuralModel>,  // Optional - only if GPU/sufficient RAM
}

impl EnsemblePredictor {
    pub fn predict(
        &self,
        recent_commands: &[Command],
        context: &Context,
        partial_input: &str,
        top_k: usize,
    ) -> Vec<Prediction> {

        // N-gram predictions (fastest, ~0.1ms)
        let ngram_preds = self.ngram.predict(
            recent_commands.last_n(3),
            partial_input
        );

        // Markov predictions with context (~1ms)
        let markov_preds = self.markov.predict(
            recent_commands.last_n(5),
            context,
            partial_input
        );

        // Neural predictions (~10-30ms, async)
        let neural_preds = self.neural.as_ref().map(|m|
            m.predict(recent_commands.last_n(10), context, partial_input)
        ).unwrap_or_default();

        // Ensemble scoring:
        // Weight: ngram 30% + markov 40% + neural 30%
        let mut scored: HashMap<String, f32> = HashMap::new();

        for pred in &ngram_preds {
            *scored.entry(pred.command.clone()).or_default() += pred.score * 0.30;
        }
        for pred in &markov_preds {
            *scored.entry(pred.command.clone()).or_default() += pred.score * 0.40;
        }
        for pred in &neural_preds {
            *scored.entry(pred.command.clone()).or_default() += pred.score * 0.30;
        }

        // Context boost: if in git repo, boost git commands
        self.apply_context_boost(&mut scored, context);

        // Sort and return top K
        let mut results: Vec<Prediction> = scored.into_iter()
            .map(|(cmd, score)| Prediction { command: cmd, score })
            .collect();

        results.sort_by(|a, b| b.score.partial_cmp(&a.score).unwrap());
        results.truncate(top_k);
        results
    }

    fn apply_context_boost(&self, scores: &mut HashMap<String, f32>, ctx: &Context) {
        // In a git repo → boost git, gh, hub commands
        if ctx.git_repo.is_some() {
            for cmd in &["git commit", "git push", "git pull", "git status"] {
                if let Some(s) = scores.get_mut(*cmd) { *s *= 1.4; }
            }
        }

        // docker-compose.yml present → boost docker-compose commands
        if ctx.has_file("docker-compose.yml") {
            if let Some(s) = scores.get_mut("docker-compose up") { *s *= 1.5; }
        }

        // Morning hours (6-9am) → boost git pull, npm install, brew update
        if ctx.hour_of_day >= 6 && ctx.hour_of_day <= 9 {
            for cmd in &["git pull", "brew update", "npm install"] {
                if let Some(s) = scores.get_mut(*cmd) { *s *= 1.2; }
            }
        }
    }
}
🖥️ Terminal UI
Bash

# What OracleShell looks like in use:

$ git add .
$ git commit -m "fix: resolve auth bug"

# User types: "git"
# OracleShell shows:

$ git █
      ├── push origin main           [92%] ← most likely next
      ├── pull                        [4%]
      ├── log --oneline               [2%]
      └── status                      [1%]

# User presses Tab → accepts "push origin main"
$ git push origin main

# After a few weeks, OracleShell learns that after your
# "git commit" you ALWAYS "git push origin main" —
# so it pre-fills it instantly at 99% confidence.

# Context-aware example:
# It's Monday morning, you just ran "git pull"
$ npm█
     └── install                    [87%] ← knows your Monday routine
📦 Tech Stack Summary
Layer	Technology
Daemon	Rust (Tokio async)
Database	SQLite (sqlx)
IPC	Unix Domain Socket
Shell Integration	Bash/Zsh/Fish hooks
N-gram Model	Custom Rust implementation
Markov Model	Custom Rust implementation
Neural Model (opt.)	Candle (Rust ML framework)
CLI	Rust (clap)
Stats UI	Ratatui
🚀 Build Phases
text

Phase 1 — Core Recording (2 weeks)
├── Shell hooks (bash + zsh)
├── Command recorder daemon
├── SQLite storage
└── Unix socket IPC

Phase 2 — N-gram Predictor (2 weeks)
├── Command tokenizer
├── Bigram + trigram model
├── Basic autocomplete integration
└── Inline ghost text in shell

Phase 3 — Markov + Context (2 weeks)
├── Context collector (git, docker, dir)
├── Context-aware Markov model
├── Temporal pattern detection
└── Context boost system

Phase 4 — Continuous Training (1 week)
├── Background training scheduler
├── Incremental model updates
└── Prediction accuracy tracking

Phase 5 — Neural + Fish Support (2 weeks)
├── Optional LSTM via Candle
├── Fish shell integration
├── Stats TUI dashboard
└── Config system
