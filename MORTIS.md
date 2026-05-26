☠️ MORTIS
The open-source "kill switch" for your entire digital life — one button, everything gone, provably
🧭 The Vision
MORTIS is the thing nobody has built because nobody wants to think about it. But every journalist, activist, abuse survivor, whistleblower, or just anyone who values sovereignty over their own data needs it. MORTIS is a cryptographically provable, multi-modal digital self-destruct system. It maintains an always-current, encrypted inventory of every digital trace of your existence across every service and device you configure. When triggered (manually, by a dead man's switch, by a specific hardware event, or by a canary signal), it executes a choreographed, provable destruction sequence: deletes local files with DoD-grade overwrite, sends account deletion requests to every service in your list, revokes OAuth tokens, wipes SSH keys, destroys password vaults, publishes a cryptographic receipt proving destruction occurred, and optionally sends a pre-written final message. The goal: leave no trace, on your terms, in your timeframe, provably.

🏗️ Why It Doesn't Exist Yet
There are account deletion services (JustDelete.me lists them). There are secure file wipers (BleachBit, Eraser). There are dead man's switches (simple email schedulers). But no tool combines:

Choreographed, sequenced multi-service deletion with dependency ordering
Cryptographic receipts proving deletion occurred
Multiple trigger mechanisms (manual, hardware, dead man's switch, canary)
Local + cloud destruction in one system
Provable, auditable destruction logs (for legal/journalistic use)
Pre-destruction data export (export first, then destroy)
🗂️ Full Directory Structure
text

mortis/
│
├── inventory/
│   ├── scanner.rs             # Discovers accounts, files, services, keys
│   ├── cataloger.rs           # Builds/maintains destruction inventory
│   ├── verifier.rs            # Periodically verifies inventory is current
│   └── models/
│       ├── local_asset.rs     # Files, folders, databases, keys
│       ├── cloud_account.rs   # Web accounts with deletion methods
│       └── credential.rs      # API tokens, SSH keys, OAuth grants
│
├── triggers/
│   ├── manual.rs              # CLI command + passphrase confirmation
│   ├── dead_mans_switch.rs    # Regular check-in required; miss = trigger
│   ├── hardware.rs            # USB key removal / specific device presence
│   ├── canary.rs              # Canary file/token: if accessed = trigger
│   └── remote.rs              # Encrypted remote trigger via signal/email
│
├── executors/
│   ├── orchestrator.rs        # Sequences and executes destruction plan
│   ├── file_destroyer.rs      # DoD 5220.22-M 7-pass overwrite + delete
│   ├── account_deleter.rs     # Automated account deletion (per service)
│   ├── key_revoker.rs         # SSH, PGP, API key revocation
│   ├── vault_destroyer.rs     # KeePass / Bitwarden local vault wiper
│   └── exporter.rs            # Pre-destruction data export (optional)
│
├── services/                  # Per-service deletion plugins
│   ├── google.rs              # Google account takeout + deletion
│   ├── github.rs              # GitHub account deletion API
│   ├── twitter.rs             # Twitter account deletion
│   ├── facebook.rs            # Facebook deletion (browser automation)
│   ├── dropbox.rs
│   ├── generic_gdpr.rs        # GDPR Article 17 "right to erasure" request
│   └── plugin_api.rs          # Community plugin interface for new services
│
├── crypto/
│   ├── receipt.rs             # Generates cryptographic destruction receipt
│   ├── signer.rs              # Signs receipt with your PGP key
│   └── timestamp.rs           # RFC 3161 trusted timestamping
│
├── canary/
│   ├── canary_file.rs         # Creates canary files/tokens in remote services
│   └── monitor.rs             # Monitors canaries for unauthorized access
│
├── final_message/
│   ├── composer.rs            # Pre-writes encrypted final messages
│   └── sender.rs              # Sends on trigger (email, Signal, dead drop)
│
├── db/
│   └── inventory.db           # SQLite: encrypted asset inventory
│
├── cli/
│   └── main.rs                # CLI interface
│
└── config/
    └── mortis.toml
⚙️ The Destruction Sequence
text

MORTIS EXECUTION SEQUENCE
════════════════════════════════════════════════════════
[T+0s]    Trigger received. Passphrase confirmed.
[T+1s]    EXPORT phase: Dump all local data to encrypted archive
[T+30s]   REVOKE phase: Revoke all API tokens, OAuth grants
[T+45s]   KEYS phase: Delete SSH keys, PGP keys from keychain
[T+60s]   VAULT phase: Overwrite + delete password vaults
[T+90s]   CLOUD phase: Submit deletion requests to all services
           → Google (GDPR Article 17 request)
           → GitHub (API deletion)
           → Twitter (API deletion)
           → Dropbox (API deletion)
           → [Custom service list...]
[T+5min]  FILES phase: 7-pass overwrite of all indexed local files
[T+15min] VERIFY phase: Re-scan to confirm deletion
[T+16min] RECEIPT phase: Generate cryptographic destruction receipt
           → Sign with PGP key
           → RFC 3161 trusted timestamp
           → Publish to: [configured dead drop / blockchain anchor]
[T+17min] MESSAGE phase: Send final message to configured recipients
[T+18min] SELF phase: Wipe MORTIS itself and its config
════════════════════════════════════════════════════════
MORTIS COMPLETE. Cryptographic receipt: a3f9b2c1...
🔐 Cryptographic Receipt Format
JSON

{
  "mortis_version": "1.0.0",
  "execution_id": "a3f9b2c1-...",
  "triggered_at": "2025-01-12T03:47:22Z",
  "trigger_type": "dead_mans_switch",
  "assets_destroyed": {
    "local_files": { "count": 14782, "total_bytes": 48293847293 },
    "cloud_accounts": { "count": 23, "deletion_requests_sent": 23 },
    "api_tokens_revoked": 47,
    "ssh_keys_deleted": 3,
    "pgp_keys_deleted": 1
  },
  "verification_hashes": {
    "pre_destruction_inventory_hash": "sha256:...",
    "post_destruction_scan_hash": "sha256:..."
  },
  "rfc3161_timestamp": "...",
  "pgp_signature": "...",
  "final_message_sent": true,
  "self_destruct_complete": true
}
📦 Tech Stack
Layer	Technology
Language	Rust
File Wiping	Custom DoD 5220.22-M implementation
Browser Automation	Playwright (for services without APIs)
Cryptography	ring crate (AES, SHA) + sequoia-pgp
Timestamping	RFC 3161 (rustls)
DB	SQLite (SQLCipher encrypted)
CLI	clap
Dead Man's Switch	Tokio scheduled tasks + SMTP
