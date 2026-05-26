🧠 DeadDrop OS
Overview
A minimal Linux-based live OS that forgets everything on shutdown, actively deceives snoopers with a panic layout, routes all traffic through Tor, and has a built-in dead drop messaging system.

Architecture Overview
text

┌─────────────────────────────────────────────────────┐
│                   DeadDrop OS                        │
├──────────────┬──────────────┬────────────────────────┤
│  Base Layer  │ Network Layer│    Application Layer    │
│  (Debian     │  (Tor +      │  (Dead Drop Messenger + │
│   Live)      │   Firewall)  │   Panic Mode + UI)      │
└──────────────┴──────────────┴────────────────────────┘
Full Component Breakdown
1. Base OS Layer
Base: Debian Live (minimal, no desktop environment by default)
Init system: systemd (stripped down)
RAM-only mode: Use tmpfs for all writes — nothing ever touches disk
Boot: GRUB with a custom splash screen, boots into a minimal TTY or lightweight WM (Openbox)
Persistence: Explicitly NONE — all state lives in RAM
Bash

# Build system using live-build
lb config \
  --distribution bookworm \
  --archive-areas "main contrib non-free" \
  --bootappend-live "boot=live components amnesia" \
  --debian-installer false \
  --memtest none \
  --win32-loader false
2. Network Layer — Tor Routing
Every single packet goes through Tor. No exceptions.

text

┌────────────┐     ┌──────────┐     ┌──────────┐     ┌───────┐
│  Any App   │────▶│iptables  │────▶│  Tor     │────▶│ Exit  │
│            │     │(redirect)│     │(SOCKS5)  │     │ Node  │
└────────────┘     └──────────┘     └──────────┘     └───────┘
iptables rules to force all traffic through Tor:

Bash

#!/bin/bash
# /etc/deadrop/firewall.sh

TOR_UID=$(id -u debian-tor)
TOR_PORT=9040
DNS_PORT=5353

# Flush existing
iptables -F
iptables -t nat -F

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow Tor process itself
iptables -A OUTPUT -m owner --uid-owner $TOR_UID -j ACCEPT

# Block everything else outbound that isn't Tor
iptables -t nat -A OUTPUT ! -o lo ! -d 127.0.0.1 \
  -p tcp -m owner ! --uid-owner $TOR_UID -j REDIRECT --to-ports $TOR_PORT

# DNS via Tor
iptables -t nat -A OUTPUT -p udp --dport 53 -j REDIRECT --to-ports $DNS_PORT

# Drop everything that escapes
iptables -A OUTPUT -j DROP
torrc config:

text

# /etc/tor/torrc
VirtualAddrNetworkIPv4 10.192.0.0/10
AutomapHostsOnResolve 1
TransPort 9040
DNSPort 5353
SocksPort 9050
3. Dead Drop Messenger
The core communication system. Messages are stored encrypted on a shared onion service. Think of a public bulletin board where only the right key reveals your message.

text

┌─────────────┐         ┌──────────────────┐         ┌─────────────┐
│   Sender    │──POST──▶│  Onion Service   │◀──GET───│  Receiver   │
│ (encrypts   │         │  (SQLite store)  │         │ (decrypts   │
│  w/ pubkey) │         │                  │         │  w/ privkey)│
└─────────────┘         └──────────────────┘         └─────────────┘
Message Schema (SQLite):

SQL

CREATE TABLE drops (
    id          TEXT PRIMARY KEY,       -- random UUID
    payload     BLOB NOT NULL,          -- PGP-encrypted message
    timestamp   INTEGER NOT NULL,       -- Unix time
    expires_at  INTEGER NOT NULL,       -- auto-delete after X hours
    hint        TEXT                    -- public unencrypted hint e.g. "for:alice"
);

-- Auto-cleanup trigger
CREATE TRIGGER auto_expire
AFTER INSERT ON drops
BEGIN
    DELETE FROM drops WHERE expires_at < strftime('%s','now');
END;
Messenger TUI (Python + Textual):

Python

# deadrop/messenger.py
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Input, ListView, ListItem, Label
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes, serialization
import sqlite3, uuid, time, requests

class DeadDropApp(App):
    CSS_PATH = "messenger.css"
    BINDINGS = [
        ("ctrl+n", "new_drop", "New Drop"),
        ("ctrl+r", "refresh", "Refresh"),
        ("ctrl+p", "panic", "PANIC"),
    ]

    def compose(self) -> ComposeResult:
        yield Header()
        yield ListView(id="drops")
        yield Input(placeholder="Enter recipient pubkey fingerprint...", id="recipient")
        yield Input(placeholder="Your message...", id="message")
        yield Footer()

    def action_new_drop(self):
        recipient_fp = self.query_one("#recipient").value
        message = self.query_one("#message").value
        encrypted = self.encrypt_message(message, recipient_fp)
        self.post_drop(encrypted)

    def encrypt_message(self, message: str, pubkey_pem: str) -> bytes:
        pubkey = serialization.load_pem_public_key(pubkey_pem.encode())
        return pubkey.encrypt(
            message.encode(),
            padding.OAEP(
                mgf=padding.MGF1(algorithm=hashes.SHA256()),
                algorithm=hashes.SHA256(),
                label=None
            )
        )

    def post_drop(self, payload: bytes):
        drop_id = str(uuid.uuid4())
        expires = int(time.time()) + (24 * 3600)  # 24h TTL
        # POST to local onion service
        requests.post(
            "http://127.0.0.1:8080/drop",
            json={"id": drop_id, "payload": payload.hex(), "expires_at": expires},
            proxies={"http": "socks5h://127.0.0.1:9050"}
        )
4. Panic Mode System
The crown jewel. One key combo → instant deception.

Python

# deadrop/panic.py
import subprocess, os, signal
from pynput import keyboard

PANIC_COMBO = {keyboard.Key.ctrl_l, keyboard.Key.alt_l, keyboard.KeyCode.from_char('p')}
current_keys = set()

DECOY_APPS = [
    ("libreoffice", ["--calc", "/etc/deadrop/decoy/budget_2024.ods"]),
    ("firefox-esr", ["https://weather.com"]),
]

def on_press(key):
    current_keys.add(key)
    if PANIC_COMBO.issubset(current_keys):
        trigger_panic()

def trigger_panic():
    # 1. Kill all sensitive processes
    subprocess.run(["pkill", "-f", "deadrop"])
    subprocess.run(["pkill", "-f", "tor"])

    # 2. Wipe RAM-resident sensitive data
    subprocess.run(["sync"])
    with open("/proc/sys/vm/drop_caches", "w") as f:
        f.write("3")

    # 3. Launch decoy applications
    for app, args in DECOY_APPS:
        subprocess.Popen([app] + args)

    # 4. Switch to decoy wallpaper and cursor theme
    subprocess.run(["xfconf-query", "-c", "xfce4-desktop",
                    "-p", "/backdrop/screen0/monitor0/image-path",
                    "-s", "/etc/deadrop/decoy/wallpaper.jpg"])

def start_panic_listener():
    with keyboard.Listener(on_press=on_press, on_release=lambda k: current_keys.discard(k)) as l:
        l.join()
5. Directory Structure
text

deaddrop-os/
├── build/
│   ├── live-build-config/
│   │   ├── auto/
│   │   │   ├── config
│   │   │   └── build
│   │   └── config/
│   │       ├── package-lists/
│   │       │   └── deadrop.list.chroot
│   │       ├── hooks/
│   │       │   └── normal/
│   │       │       ├── 0001-tor-setup.hook.chroot
│   │       │       ├── 0002-firewall.hook.chroot
│   │       │       └── 0003-deadrop-install.hook.chroot
│   │       └── includes.chroot/
│   │           └── etc/
│   │               ├── tor/torrc
│   │               ├── deadrop/
│   │               │   ├── firewall.sh
│   │               │   ├── decoy/
│   │               │   │   ├── wallpaper.jpg
│   │               │   │   └── budget_2024.ods
│   │               │   └── config.toml
│   │               └── systemd/system/
│   │                   ├── deadrop-messenger.service
│   │                   ├── deadrop-panic.service
│   │                   └── deadrop-onion.service
├── src/
│   ├── messenger/          # Python/Textual TUI
│   ├── panic/              # Panic mode daemon
│   ├── onion-server/       # Go HTTP server (onion service backend)
│   └── firewall/           # iptables management
├── tests/
└── README.md
6. Onion Service Backend (Go)
Go

// src/onion-server/main.go
package main

import (
    "database/sql"
    "encoding/json"
    "log"
    "net/http"
    "time"
    _ "github.com/mattn/go-sqlite3"
    "github.com/cretz/bine/tor"
)

type Drop struct {
    ID        string `json:"id"`
    Payload   string `json:"payload"`
    ExpiresAt int64  `json:"expires_at"`
    Hint      string `json:"hint"`
}

func main() {
    // Start embedded Tor
    t, _ := tor.Start(nil, nil)
    defer t.Close()

    // Create onion service on port 8080
    onion, _ := t.Listen(nil, &tor.ListenConf{
        RemotePorts: []int{80},
        LocalPort:   8080,
    })
    defer onion.Close()

    log.Printf("Onion address: %s.onion", onion.ID)

    db, _ := sql.Open("sqlite3", ":memory:") // RAM only!
    initDB(db)

    mux := http.NewServeMux()
    mux.HandleFunc("/drop", dropHandler(db))
    mux.HandleFunc("/drops", listHandler(db))

    http.Serve(onion, mux)
}

func dropHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if r.Method == "POST" {
            var drop Drop
            json.NewDecoder(r.Body).Decode(&drop)
            db.Exec(`INSERT INTO drops VALUES (?,?,?,?)`,
                drop.ID, drop.Payload,
                time.Now().Unix(), drop.ExpiresAt)
            w.WriteHeader(201)
        }
    }
}
7. Build & Release Pipeline
YAML

# .github/workflows/build-iso.yml
name: Build DeadDrop OS ISO
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install live-build
        run: sudo apt-get install -y live-build
      - name: Build ISO
        run: |
          cd build/live-build-config
          sudo lb build
      - name: Upload ISO artifact
        uses: actions/upload-artifact@v3
        with:
          name: deaddrop-os.iso
          path: build/live-build-config/*.iso
8. Development Roadmap
text

Phase 1 (Weeks 1-4): Base OS
├── Set up live-build config
├── Tor routing + iptables firewall
├── Boot to TTY with Openbox
└── Verify zero-disk-write on boot

Phase 2 (Weeks 5-8): Dead Drop Messenger
├── SQLite schema + onion server (Go)
├── Key generation + PGP encryption
├── Textual TUI for messaging
└── End-to-end encryption tests

Phase 3 (Weeks 9-12): Panic Mode
├── Key combo listener daemon
├── Process kill + RAM wipe
├── Decoy app launcher
└── Decoy wallpaper/session swap

Phase 4 (Weeks 13-16): Polish & Release
├── ISO signing + verification
├── Security audit
├── Documentation
└── Release on GitHub + Tor hidden service
