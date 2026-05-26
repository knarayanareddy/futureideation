🧬 CodeDNA
A local AI that learns YOUR coding style from your git history and acts as a "style immune system" — rejecting code that isn't you
🧭 The Vision
Every developer has a style — not just formatting, but deep patterns: how they name variables, how they structure conditionals, how long their functions tend to be, how they comment, what abstractions they reach for, how they handle errors, what libraries they prefer, how they think about architecture. CodeDNA analyzes your entire git history, builds a deep stylometric fingerprint of how YOU write code, and then acts as a living immune system for your codebase. It runs in your IDE/terminal and flags code that doesn't sound like you — whether it's AI-generated slop, a copy-paste from Stack Overflow, or a collaborator's conflicting style. It doesn't just lint. It understands your voice as a programmer and tells you when something is out of character — at the function level, the file level, the PR level. Over time, it builds a chronological style evolution graph: how has your coding style changed over 5 years? When did you start preferring functional patterns? When did you abandon global state?

🏗️ Why It Doesn't Exist Yet
Linters check style rules you define. Copilot generates code in your style superficially. But no tool:

Learns YOUR specific deep stylometric fingerprint from YOUR git history
Acts as a passive immune system (flags foreign code automatically)
Tracks how your style has evolved over time
Can tell you "this function was definitely not written by you" with high confidence
Works locally with zero cloud
Treats style as a biological fingerprint, not a ruleset
🗂️ Full Directory Structure
text

codedna/
│
├── analyzer/
│   ├── git_parser.py          # Walks git history, extracts author commits
│   ├── ast_extractor.py       # Language-agnostic AST feature extraction
│   ├── feature_builder.py     # Builds feature vectors from AST + text
│   └── supported_langs/
│       ├── python_features.py
│       ├── rust_features.py
│       ├── javascript_features.py
│       ├── go_features.py
│       └── typescript_features.py
│
├── fingerprint/
│   ├── builder.py             # Builds StyleDNA from feature vectors
│   ├── clusterer.py           # UMAP + HDBSCAN clustering of style space
│   ├── evolver.py             # Tracks fingerprint changes over time
│   └── comparator.py          # Compares two code fragments' style
│
├── immune_system/
│   ├── scanner.rs             # Fast Rust binary: scans code in real-time
│   ├── scorer.py              # Scores "does this code match my DNA?"
│   ├── detector.py            # Detects foreign code regions in a file
│   └── explainer.py           # Explains WHY something is flagged
│
├── evolution/
│   ├── timeline.py            # Builds style evolution timeline
│   ├── shift_detector.py      # Detects when style shifted significantly
│   └── report.py              # "Your coding style history" report
│
├── integrations/
│   ├── vscode_extension/      # VS Code extension (TypeScript)
│   ├── vim_plugin/            # Vim/Neovim plugin (Lua)
│   ├── git_hook.rs            # Pre-commit hook
│   └── cli.py                 # Standalone CLI
│
├── db/
│   ├── fingerprints.db        # SQLite: style vectors, history
│   └── schema.sql
│
└── api/
    ├── server.py              # FastAPI (IDE extensions call this)
    └── routes/
        ├── analyze.py
        ├── score.py
        └── evolution.py
🧬 The Style Feature Vector
Python

@dataclass
class StyleFeatures:
    """
    ~200-dimensional feature vector capturing coding style
    Extracted from AST + raw text analysis
    """
    # === NAMING ===
    avg_name_length: float              # Variable/function name lengths
    snake_case_ratio: float             # snake_case vs camelCase preference
    abbreviation_ratio: float           # "btn" vs "button" tendency
    descriptiveness: float              # avg token information content

    # === STRUCTURE ===
    avg_function_length: float          # Lines per function
    avg_file_length: float              # Lines per file
    nesting_depth_avg: float            # Average nesting depth
    early_return_ratio: float           # Guard clause preference
    single_vs_multi_return: float       # One return vs multiple
    linear_vs_nested_ratio: float       # Flat code vs deeply nested

    # === PATTERNS ===
    oop_vs_functional: float            # Class usage vs function-first
    mutation_ratio: float               # Mutable vs immutable patterns
    exception_style: float              # try/catch vs Result/Option
    abstraction_eagerness: float        # How quickly they abstract things
    comment_density: float              # Comments per line of code
    comment_style: float                # Inline vs block vs docstring ratio

    # === DEPENDENCIES ===
    import_count_avg: float             # How many imports per file
    stdlib_preference: float            # stdlib vs third-party ratio
    preferred_lib_signature: List[str]  # Most used libraries (hashed)

    # === TEMPORAL ===
    commit_size_avg: float              # Avg lines per commit
    refactor_frequency: float           # How often they go back and change
    test_coverage_correlation: float    # How correlated tests are with features
🛡️ The Immune System in Action
text

$ git commit -m "add payment handler"
🧬 CodeDNA — Style Analysis
════════════════════════════════════════════════════
Scanning: src/payments/handler.py

OVERALL MATCH: 34% ⚠️  FOREIGN CODE DETECTED

Line 12-47: process_payment()
  Similarity: 18% 🔴
  This function reads like GPT-4 output. Specifically:
  ↳ You never use `isinstance()` — you prefer duck typing
  ↳ Your functions average 12 lines. This is 41 lines.
  ↳ You always use early returns. This has 3 nested if blocks.
  ↳ Variable name `temp_result` is not your style.
  ↳ Comment density: 3x higher than your baseline.

Line 52-61: validate_card()
  Similarity: 78% ✅ — This sounds like you.

Line 63-89: format_receipt()
  Similarity: 31% 🔴
  ↳ Matches Stack Overflow answer pattern (high specificity).
  ↳ You never use f-string concatenation for multiline strings.

Commit blocked. Review flagged sections? [y/N]
════════════════════════════════════════════════════
📦 Tech Stack
Layer	Technology
Language	Python + Rust
Git Parsing	GitPython
AST Analysis	tree-sitter (multi-language)
ML/Clustering	scikit-learn, UMAP, HDBSCAN
Embeddings	CodeBERT (local)
Fast Scanner	Rust binary (ripgrep-style speed)
IDE Extension	TypeScript (VS Code)
Vim Plugin	Lua (Neovim)
API	FastAPI
Database	SQLite
