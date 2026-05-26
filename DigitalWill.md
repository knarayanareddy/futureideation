🪦 DigitalWill
Overview
A self-hosted, encrypted dead man's switch. Check in regularly. Miss your check-in, and pre-configured actions fire automatically — send emails, publish files, post to social media, wipe data.

Architecture Overview
text

┌────────────────────────────────────────────────────────────┐
│                      DigitalWill                            │
├──────────────┬─────────────────┬───────────────────────────┤
│  Scheduler   │  Action Engine  │  Encryption & Storage      │
│  (check-in   │  (email, post,  │  (SQLite + AES-256        │
│   timers)    │   publish, wipe)│   + PGP)                   │
└──────────────┴─────────────────┴───────────────────────────┘
Full Component Breakdown
1. Project Structure
text

digitalwill/
├── cmd/
│   ├── will/
│   │   └── main.go          # CLI entry point
│   └── willd/
│       └── main.go          # Background daemon
├── internal/
│   ├── checkin/
│   │   ├── checkin.go       # Check-in logic
│   │   └── scheduler.go     # Timer management
│   ├── actions/
│   │   ├── action.go        # Action interface
│   │   ├── email.go         # SMTP email action
│   │   ├── publish.go       # File publish action
│   │   ├── social.go        # Social media post
│   │   ├── wipe.go          # Data wipe action
│   │   └── webhook.go       # Generic webhook
│   ├── crypto/
│   │   ├── encrypt.go       # AES-256-GCM encryption
│   │   └── pgp.go           # PGP operations
│   ├── storage/
│   │   └── db.go            # SQLite operations
│   ├── config/
│   │   └── config.go        # TOML config management
│   └── tui/
│       └── tui.go           # Terminal UI
├── config.example.toml
├── go.mod
└── README.md
2. Database Schema
SQL

-- schema.sql

CREATE TABLE wills (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    description     TEXT,
    check_in_interval INTEGER NOT NULL, -- seconds
    grace_period    INTEGER DEFAULT 3600, -- seconds after missed check-in
    last_check_in   INTEGER NOT NULL,
    status          TEXT DEFAULT 'active', -- active, triggered, paused, disabled
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL
);

CREATE TABLE actions (
    id          TEXT PRIMARY KEY,
    will_id     TEXT NOT NULL REFERENCES wills(id),
    type        TEXT NOT NULL, -- email, publish, social, wipe, webhook
    priority    INTEGER DEFAULT 0, -- execution order
    delay       INTEGER DEFAULT 0, -- delay after trigger (seconds)
    config_enc  BLOB NOT NULL,     -- AES-256-GCM encrypted JSON config
    status      TEXT DEFAULT 'pending', -- pending, executed, failed
    executed_at INTEGER,
    FOREIGN KEY (will_id) REFERENCES wills(id)
);

CREATE TABLE check_ins (
    id          TEXT PRIMARY KEY,
    will_id     TEXT NOT NULL,
    timestamp   INTEGER NOT NULL,
    method      TEXT,   -- cli, web, email, sms
    ip_hash     TEXT    -- hashed for minimal logging
);

CREATE TABLE execution_log (
    id          TEXT PRIMARY KEY,
    will_id     TEXT NOT NULL,
    action_id   TEXT NOT NULL,
    timestamp   INTEGER NOT NULL,
    result      TEXT,   -- success, failure
    message     TEXT
);

CREATE TABLE secrets (
    key         TEXT PRIMARY KEY,
    value_enc   BLOB NOT NULL,  -- encrypted value
    created_at  INTEGER NOT NULL
);
3. Core Will Engine (Go)
Go

// internal/checkin/scheduler.go
package checkin

import (
    "context"
    "log"
    "time"
    "digitalwill/internal/actions"
    "digitalwill/internal/storage"
)

type Scheduler struct {
    db      *storage.DB
    engine  *actions.Engine
    tickers map[string]*time.Ticker
}

func NewScheduler(db *storage.DB, engine *actions.Engine) *Scheduler {
    return &Scheduler{
        db:      db,
        engine:  engine,
        tickers: make(map[string]*time.Ticker),
    }
}

func (s *Scheduler) Start(ctx context.Context) {
    // Check every minute if any will has been missed
    ticker := time.NewTicker(60 * time.Second)
    defer ticker.Stop()

    log.Println("🪦 DigitalWill scheduler started")

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            s.checkAllWills()
        }
    }
}

func (s *Scheduler) checkAllWills() {
    wills, err := s.db.GetActiveWills()
    if err != nil {
        log.Printf("Error fetching wills: %v", err)
        return
    }

    now := time.Now().Unix()

    for _, will := range wills {
        deadline := will.LastCheckIn + will.CheckInInterval + will.GracePeriod

        if now > deadline {
            log.Printf("🚨 Will '%s' triggered! Executing actions...", will.Name)
            s.db.UpdateWillStatus(will.ID, "triggered")
            go s.engine.ExecuteWill(will)
        } else {
            remaining := deadline - now
            log.Printf("✅ Will '%s' OK. Deadline in %ds", will.Name, remaining)
        }
    }
}

func (s *Scheduler) CheckIn(willID string, method string) error {
    return s.db.RecordCheckIn(willID, method, time.Now().Unix())
}
4. Action Engine & Action Types
Go

// internal/actions/engine.go
package actions

import (
    "log"
    "sort"
    "time"
    "digitalwill/internal/storage"
)

type Engine struct {
    db *storage.DB
}

type ActionConfig struct {
    Type    string                 `json:"type"`
    Delay   int                    `json:"delay"`
    Params  map[string]interface{} `json:"params"`
}

func (e *Engine) ExecuteWill(will storage.Will) {
    actions, _ := e.db.GetActionsForWill(will.ID)

    // Sort by priority
    sort.Slice(actions, func(i, j int) bool {
        return actions[i].Priority < actions[j].Priority
    })

    for _, action := range actions {
        // Respect delay
        if action.Delay > 0 {
            time.Sleep(time.Duration(action.Delay) * time.Second)
        }

        var result string
        var err error

        switch action.Type {
        case "email":
            err = ExecuteEmail(action.Config)
        case "publish":
            err = ExecutePublish(action.Config)
        case "social":
            err = ExecuteSocial(action.Config)
        case "wipe":
            err = ExecuteWipe(action.Config)
        case "webhook":
            err = ExecuteWebhook(action.Config)
        }

        if err != nil {
            result = "failure: " + err.Error()
            log.Printf("❌ Action %s failed: %v", action.Type, err)
        } else {
            result = "success"
            log.Printf("✅ Action %s executed successfully", action.Type)
        }

        e.db.LogExecution(will.ID, action.ID, result)
    }
}
Go

// internal/actions/email.go
package actions

import (
    "crypto/tls"
    "fmt"
    "net/smtp"
)

type EmailConfig struct {
    SMTPHost   string   `json:"smtp_host"`
    SMTPPort   int      `json:"smtp_port"`
    Username   string   `json:"username"`
    Password   string   `json:"password"`
    From       string   `json:"from"`
    To         []string `json:"to"`
    Subject    string   `json:"subject"`
    Body       string   `json:"body"`
    Attachment string   `json:"attachment_path"` // optional file
}

func ExecuteEmail(config map[string]interface{}) error {
    cfg := parseEmailConfig(config)

    tlsConfig := &tls.Config{
        InsecureSkipVerify: false,
        ServerName:         cfg.SMTPHost,
    }

    conn, err := tls.Dial("tcp",
        fmt.Sprintf("%s:%d", cfg.SMTPHost, cfg.SMTPPort),
        tlsConfig)
    if err != nil {
        return err
    }

    client, err := smtp.NewClient(conn, cfg.SMTPHost)
    if err != nil {
        return err
    }

    auth := smtp.PlainAuth("", cfg.Username, cfg.Password, cfg.SMTPHost)
    client.Auth(auth)
    client.Mail(cfg.From)

    for _, recipient := range cfg.To {
        client.Rcpt(recipient)
    }

    writer, _ := client.Data()
    message := buildMIMEMessage(cfg)
    writer.Write([]byte(message))
    writer.Close()
    client.Quit()

    return nil
}
Go

// internal/actions/social.go
package actions

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
)

type SocialConfig struct {
    Platform    string `json:"platform"`  // mastodon, bluesky, nostr
    APIEndpoint string `json:"api_endpoint"`
    Token       string `json:"token"`
    Message     string `json:"message"`
    FileToPost  string `json:"file_path"` // optional
}

func ExecuteSocial(config map[string]interface{}) error {
    cfg := parseSocialConfig(config)

    switch cfg.Platform {
    case "mastodon":
        return postToMastodon(cfg)
    case "bluesky":
        return postToBluesky(cfg)
    case "nostr":
        return postToNostr(cfg)
    }
    return fmt.Errorf("unknown platform: %s", cfg.Platform)
}

func postToMastodon(cfg SocialConfig) error {
    payload := map[string]string{
        "status":     cfg.Message,
        "visibility": "public",
    }
    body, _ := json.Marshal(payload)

    req, _ := http.NewRequest("POST",
        cfg.APIEndpoint+"/api/v1/statuses",
        bytes.NewBuffer(body))
    req.Header.Set("Authorization", "Bearer "+cfg.Token)
    req.Header.Set("Content-Type", "application/json")

    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    return nil
}
Go

// internal/actions/wipe.go
package actions

import (
    "os"
    "path/filepath"
    "crypto/rand"
)

type WipeConfig struct {
    Paths      []string `json:"paths"`    // files/dirs to wipe
    Passes     int      `json:"passes"`   // overwrite passes (DoD = 3, Gutmann = 35)
    ShredFiles bool     `json:"shred"`    // overwrite before delete
}

func ExecuteWipe(config map[string]interface{}) error {
    cfg := parseWipeConfig(config)

    for _, path := range cfg.Paths {
        if err := secureDelete(path, cfg.Passes); err != nil {
            return err
        }
    }
    return nil
}

func secureDelete(path string, passes int) error {
    info, err := os.Stat(path)
    if err != nil {
        return err
    }

    if info.IsDir() {
        return filepath.Walk(path, func(p string, fi os.FileInfo, err error) error {
            if !fi.IsDir() {
                return shredFile(p, passes)
            }
            return nil
        })
    }
    return shredFile(path, passes)
}

func shredFile(path string, passes int) error {
    file, err := os.OpenFile(path, os.O_WRONLY, 0)
    if err != nil {
        return err
    }
    defer file.Close()

    info, _ := file.Stat()
    size := info.Size()

    // Overwrite with random data
    for i := 0; i < passes; i++ {
        file.Seek(0, 0)
        buf := make([]byte, size)
        rand.Read(buf)
        file.Write(buf)
        file.Sync()
    }

    file.Close()
    return os.Remove(path)
}
5. Encryption Layer
Go

// internal/crypto/encrypt.go
package crypto

import (
    "crypto/aes"
    "crypto/cipher"
    "crypto/rand"
    "crypto/sha256"
    "encoding/base64"
    "io"
    "golang.org/x/crypto/pbkdf2"
)

const (
    saltSize   = 32
    iterations = 310000  // OWASP recommended for PBKDF2-SHA256
    keySize    = 32      // AES-256
)

type Encryptor struct {
    masterPassword []byte
}

func NewEncryptor(password string) *Encryptor {
    return &Encryptor{masterPassword: []byte(password)}
}

func (e *Encryptor) Encrypt(plaintext []byte) (string, error) {
    salt := make([]byte, saltSize)
    io.ReadFull(rand.Reader, salt)

    key := pbkdf2.Key(e.masterPassword, salt, iterations, keySize, sha256.New)

    block, err := aes.NewCipher(key)
    if err != nil {
        return "", err
    }

    gcm, err := cipher.NewGCM(block)
    if err != nil {
        return "", err
    }

    nonce := make([]byte, gcm.NonceSize())
    io.ReadFull(rand.Reader, nonce)

    ciphertext := gcm.Seal(nonce, nonce, plaintext, nil)
    result := append(salt, ciphertext...)

    return base64.StdEncoding.EncodeToString(result), nil
}

func (e *Encryptor) Decrypt(encoded string) ([]byte, error) {
    data, err := base64.StdEncoding.DecodeString(encoded)
    if err != nil {
        return nil, err
    }

    salt := data[:saltSize]
    ciphertext := data[saltSize:]

    key := pbkdf2.Key(e.masterPassword, salt, iterations, keySize, sha256.New)

    block, _ := aes.NewCipher(key)
    gcm, _ := cipher.NewGCM(block)

    nonce := ciphertext[:gcm.NonceSize()]
    ciphertext = ciphertext[gcm.NonceSize():]

    return gcm.Open(nil, nonce, ciphertext, nil)
}
6. Configuration File (TOML)
toml

# config.toml — all sensitive values are encrypted at rest

[master]
# This password encrypts all action configs
# Set via env var DIGITALWILL_PASSWORD, never stored here
password_env = "DIGITALWILL_PASSWORD"
data_dir = "~/.digitalwill"

[[wills]]
id = "my-main-will"
name = "My Digital Will"
description = "Release documents if I go missing"
check_in_interval = "7d"   # Must check in every 7 days
grace_period = "24h"       # 24 hours after missed check-in

  [[wills.actions]]
  type = "email"
  priority = 1
  delay = "0h"
  [wills.actions.params]
    to = ["lawyer@example.com", "friend@example.com"]
    subject = "Important: Please read this"
    body_file = "~/.digitalwill/letters/main_letter.txt"
    attachment = "~/.digitalwill/docs/instructions.pdf"

  [[wills.actions]]
  type = "publish"
  priority = 2
  delay = "1h"  # Wait 1 hour before publishing
  [wills.actions.params]
    file = "~/.digitalwill/docs/evidence.zip"
    destinations = ["ipfs", "0x0.st"]  # Upload to IPFS + temp host

  [[wills.actions]]
  type = "social"
  priority = 3
  delay = "2h"
  [wills.actions.params]
    platform = "mastodon"
    api_endpoint = "https://mastodon.social"
    token_env = "MASTODON_TOKEN"
    message_file = "~/.digitalwill/letters/public_statement.txt"

  [[wills.actions]]
  type = "wipe"
  priority = 4
  delay = "48h"  # Wipe 48h after trigger
  [wills.actions.params]
    paths = ["~/.ssh", "~/.gnupg", "~/sensitive/"]
    passes = 3
7. CLI Interface
Bash

# Usage examples

# Initial setup
will init

# Create a new will interactively
will create

# Check in (resets the timer)
will checkin

# Check in via one-liner (for cron/alias)
will checkin --will my-main-will

# View status of all wills
will status

# Test an action without triggering (dry run)
will test --will my-main-will --action email

# Pause/resume a will
will pause my-main-will
will resume my-main-will

# Start the background daemon
willd start
willd stop
willd status
Go

// cmd/will/main.go
package main

import (
    "github.com/spf13/cobra"
    "digitalwill/internal/checkin"
    "digitalwill/internal/storage"
)

var rootCmd = &cobra.Command{
    Use:   "will",
    Short: "DigitalWill — Your encrypted dead man's switch",
}

var checkinCmd = &cobra.Command{
    Use:   "checkin",
    Short: "Check in to reset your will timer",
    RunE: func(cmd *cobra.Command, args []string) error {
        willID, _ := cmd.Flags().GetString("will")
        db, _ := storage.Open()
        scheduler := checkin.NewScheduler(db, nil)
        err := scheduler.CheckIn(willID, "cli")
        if err == nil {
            fmt.Println("✅ Checked in successfully. Timer reset.")
        }
        return err
    },
}

var statusCmd = &cobra.Command{
    Use:   "status",
    Short: "Show status of all wills",
    RunE: func(cmd *cobra.Command, args []string) error {
        db, _ := storage.Open()
        wills, _ := db.GetAllWills()
        for _, w := range wills {
            remaining := w.Deadline() - time.Now().Unix()
            fmt.Printf("%-20s | %-10s | Next deadline in: %s\n",
                w.Name, w.Status, formatDuration(remaining))
        }
        return nil
    },
}

func init() {
    checkinCmd.Flags().String("will", "", "Will ID to check in")
    rootCmd.AddCommand(checkinCmd, statusCmd)
}

func main() {
    rootCmd.Execute()
}
8. Web Check-in Interface (Minimal)
Go

// For users who want to check in via browser (served locally)
// internal/web/server.go
package web

import (
    "net/http"
    "html/template"
)

const checkInHTML = `
<!DOCTYPE html>
<html>
<head><title>DigitalWill Check-In</title></head>
<body style="font-family:monospace;max-width:400px;margin:100px auto;text-align:center">
    <h2>🪦 DigitalWill</h2>
    <p>Click below to confirm you are alive and reset your timers.</p>
    <form method="POST" action="/checkin">
        <input type="hidden" name="token" value="{{.Token}}">
        <button type="submit"
            style="padding:20px 40px;font-size:18px;cursor:pointer">
            ✅ I AM ALIVE
        </button>
    </form>
    <p style="color:gray;font-size:12px">Next check-in due: {{.NextDeadline}}</p>
</body>
</html>
`
9. Development Roadmap
text

Phase 1 (Weeks 1-3): Core Engine
├── SQLite schema + CRUD operations
├── AES-256-GCM encryption layer
├── Check-in logic + timer scheduler
└── CLI (checkin, status, pause)

Phase 2 (Weeks 4-6): Action Engine
├── Email action (SMTP)
├── File publish action (IPFS + HTTP upload)
├── Wipe action (secure delete)
└── Webhook action

Phase 3 (Weeks 7-9): Social + Web
├── Mastodon integration
├── Bluesky/AT Protocol integration
├── Local web check-in interface
└── Systemd daemon unit file

Phase 4 (Weeks 10-12): Polish
├── TOML config validation
├── Dry-run mode for all actions
├── Notification system (warn before deadline)
└── Full documentation + release
