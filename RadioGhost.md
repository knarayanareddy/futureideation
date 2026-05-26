Overview
A terminal-native, anonymous, ephemeral internet radio station creator and listener. Spin up a radio station in one command, share an onion address, listeners tune in via terminal.

Architecture Overview
text

┌──────────────────────────────────────────────────────────────┐
│                        RadioGhost                             │
├─────────────────┬────────────────────┬────────────────────────┤
│  Broadcaster    │   Stream Server     │      Listener          │
│  (audio input   │  (Tor hidden svc   │  (terminal player      │
│   + encoding)   │   + Icecast-like)  │   via onion addr)      │
└─────────────────┴────────────────────┴────────────────────────┘
Full Component Breakdown
1. Core Audio Pipeline
text

Microphone/File/Playlist
        │
        ▼
   FFmpeg (encode to Opus/MP3)
        │
        ▼
   GStreamer pipeline
        │
        ▼
   Internal streaming server
        │
        ▼
   Tor Hidden Service (port 8000)
        │
        ▼
   Listener terminal client
2. Project Structure
text

radioghost/
├── src/
│   ├── main.rs                 # CLI entry point
│   ├── broadcaster/
│   │   ├── mod.rs
│   │   ├── audio_capture.rs    # mic/file/playlist input
│   │   ├── encoder.rs          # Opus encoding via libopus
│   │   └── stream_server.rs    # HTTP audio stream server
│   ├── listener/
│   │   ├── mod.rs
│   │   ├── decoder.rs          # decode incoming Opus stream
│   │   └── player.rs           # CPAL audio playback
│   ├── tor/
│   │   ├── mod.rs
│   │   ├── hidden_service.rs   # create + manage onion service
│   │   └── client.rs           # connect via SOCKS5
│   ├── tui/
│   │   ├── mod.rs
│   │   ├── broadcaster_ui.rs   # Ratatui broadcaster interface
│   │   └── listener_ui.rs      # Ratatui listener interface
│   └── config.rs
├── Cargo.toml
└── README.md
3. Cargo.toml Dependencies
toml

[package]
name = "radioghost"
version = "0.1.0"
edition = "2021"

[dependencies]
# TUI
ratatui = "0.26"
crossterm = "0.27"

# Audio
cpal = "0.15"              # Cross-platform audio I/O
opus = "0.3"               # Opus codec bindings
symphonia = "0.5"          # Audio decoding (MP3, FLAC, etc.)
rodio = "0.17"             # Audio playback

# Networking
tokio = { version = "1", features = ["full"] }
hyper = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["socks"] }

# Tor
arti-client = "0.10"       # Pure Rust Tor client (NO system Tor needed!)
tor-rtcompat = "0.10"

# Utilities
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
clap = { version = "4", features = ["derive"] }
tracing = "0.1"
tracing-subscriber = "0.3"
4. Broadcaster Core (Rust)
Rust

// src/broadcaster/stream_server.rs
use tokio::net::TcpListener;
use tokio::sync::broadcast;
use hyper::{Body, Request, Response, Server};
use hyper::service::{make_service_fn, service_fn};
use std::sync::Arc;
use std::convert::Infallible;

pub struct StreamServer {
    tx: broadcast::Sender<Vec<u8>>,
    port: u16,
    station_name: String,
    description: String,
}

impl StreamServer {
    pub fn new(port: u16, name: &str, desc: &str) -> Self {
        let (tx, _) = broadcast::channel(1024);
        StreamServer {
            tx,
            port,
            station_name: name.to_string(),
            description: desc.to_string(),
        }
    }

    pub async fn start(&self) -> anyhow::Result<()> {
        let tx = self.tx.clone();
        let name = self.station_name.clone();

        let make_svc = make_service_fn(move |_conn| {
            let tx = tx.clone();
            let name = name.clone();
            async move {
                Ok::<_, Infallible>(service_fn(move |req: Request<Body>| {
                    let tx = tx.clone();
                    let name = name.clone();
                    async move {
                        match req.uri().path() {
                            "/stream" => {
                                // Subscribe to audio broadcast channel
                                let mut rx = tx.subscribe();
                                let stream = async_stream::stream! {
                                    while let Ok(chunk) = rx.recv().await {
                                        yield Ok::<_, std::io::Error>(
                                            bytes::Bytes::from(chunk)
                                        );
                                    }
                                };
                                Ok::<_, Infallible>(
                                    Response::builder()
                                        .header("Content-Type", "audio/ogg")
                                        .header("X-Station-Name", &name)
                                        .header("icy-name", &name)
                                        .header("icy-metaint", "0")
                                        .body(Body::wrap_stream(stream))
                                        .unwrap()
                                )
                            },
                            "/info" => {
                                let info = serde_json::json!({
                                    "name": name,
                                    "codec": "opus",
                                    "bitrate": 128,
                                    "status": "live"
                                });
                                Ok::<_, Infallible>(
                                    Response::new(Body::from(info.to_string()))
                                )
                            },
                            _ => Ok::<_, Infallible>(
                                Response::builder()
                                    .status(404)
                                    .body(Body::from("not found"))
                                    .unwrap()
                            )
                        }
                    }
                }))
            }
        });

        let addr = ([127, 0, 0, 1], self.port).into();
        Server::bind(&addr).serve(make_svc).await?;
        Ok(())
    }

    pub fn broadcast_chunk(&self, data: Vec<u8>) {
        let _ = self.tx.send(data);
    }
}
5. Tor Hidden Service via Arti (Pure Rust)
Rust

// src/tor/hidden_service.rs
use arti_client::{TorClient, TorClientConfig};
use tor_rtcompat::PreferredRuntime;

pub struct OnionService {
    pub address: String,
    local_port: u16,
}

impl OnionService {
    pub async fn create(local_port: u16) -> anyhow::Result<Self> {
        let config = TorClientConfig::default();
        let client = TorClient::create_bootstrapped(config).await?;

        // Create ephemeral hidden service
        let svc_config = arti_client::onion_service::OnionServiceConfigBuilder::default()
            .build()?;

        let (svc, request_stream) = client.launch_onion_service(svc_config)?;
        let address = format!("{}.onion", svc.onion_name());

        // Forward requests from onion service to local server
        tokio::spawn(async move {
            // forward onion connections to localhost:local_port
        });

        Ok(OnionService { address, local_port })
    }
}
6. Audio Capture & Encoding
Rust

// src/broadcaster/audio_capture.rs
use cpal::traits::{DeviceTrait, HostTrait, StreamTrait};
use opus::{Encoder, Application, Channels};
use std::sync::mpsc;

pub enum AudioSource {
    Microphone,
    File(String),
    Playlist(Vec<String>),
}

pub struct AudioCapture {
    source: AudioSource,
    tx: mpsc::Sender<Vec<u8>>,
}

impl AudioCapture {
    pub fn start_microphone(tx: mpsc::Sender<Vec<u8>>) {
        let host = cpal::default_host();
        let device = host.default_input_device()
            .expect("No input device found");

        let config = device.default_input_config().unwrap();
        let sample_rate = config.sample_rate().0;

        // Opus encoder: 48kHz, stereo, VOIP optimized
        let mut encoder = Encoder::new(
            48000,
            Channels::Stereo,
            Application::Audio  // Use Audio mode for music
        ).unwrap();

        let stream = device.build_input_stream(
            &config.into(),
            move |data: &[f32], _| {
                // Convert to i16 for Opus
                let samples: Vec<i16> = data.iter()
                    .map(|&s| (s * i16::MAX as f32) as i16)
                    .collect();

                // Encode frame (960 samples @ 48kHz = 20ms)
                if samples.len() >= 960 {
                    let mut output = vec![0u8; 4000];
                    if let Ok(len) = encoder.encode(&samples[..960], &mut output) {
                        output.truncate(len);
                        let _ = tx.send(output);
                    }
                }
            },
            |err| eprintln!("Stream error: {}", err),
            None
        ).unwrap();

        stream.play().unwrap();
    }
}
7. Broadcaster TUI (Ratatui)
Rust

// src/tui/broadcaster_ui.rs
use ratatui::{
    layout::{Constraint, Direction, Layout},
    style::{Color, Modifier, Style},
    text::{Line, Span},
    widgets::{Block, Borders, Gauge, List, ListItem, Paragraph},
    Frame,
};

pub struct BroadcasterState {
    pub station_name: String,
    pub onion_address: String,
    pub listener_count: usize,
    pub volume_level: f64,
    pub is_live: bool,
    pub log_messages: Vec<String>,
    pub uptime_seconds: u64,
}

pub fn render_broadcaster(f: &mut Frame, state: &BroadcasterState) {
    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Length(3),   // Header
            Constraint::Length(5),   // Station info
            Constraint::Length(3),   // Volume meter
            Constraint::Min(0),      // Log
            Constraint::Length(3),   // Footer
        ])
        .split(f.size());

    // Header
    let header = Paragraph::new(
        Line::from(vec![
            Span::styled("📡 RADIOGHOST ", Style::default()
                .fg(Color::Red)
                .add_modifier(Modifier::BOLD)),
            Span::styled(
                if state.is_live { "● LIVE" } else { "○ OFFLINE" },
                Style::default().fg(if state.is_live { Color::Green } else { Color::Red })
            ),
        ])
    ).block(Block::default().borders(Borders::ALL));
    f.render_widget(header, chunks[0]);

    // Station info
    let info_text = vec![
        Line::from(format!("🎙 Station: {}", state.station_name)),
        Line::from(format!("🧅 Address: {}", state.onion_address)),
        Line::from(format!("👥 Listeners: {}", state.listener_count)),
        Line::from(format!("⏱  Uptime: {}s", state.uptime_seconds)),
    ];
    let info = Paragraph::new(info_text)
        .block(Block::default().title("Station Info").borders(Borders::ALL));
    f.render_widget(info, chunks[1]);

    // Volume meter
    let gauge = Gauge::default()
        .block(Block::default().title("Audio Level").borders(Borders::ALL))
        .gauge_style(Style::default().fg(Color::Green))
        .percent(state.volume_level as u16);
    f.render_widget(gauge, chunks[2]);

    // Log
    let log_items: Vec<ListItem> = state.log_messages.iter()
        .map(|m| ListItem::new(m.as_str()))
        .collect();
    let log = List::new(log_items)
        .block(Block::default().title("Log").borders(Borders::ALL));
    f.render_widget(log, chunks[3]);

    // Footer controls
    let footer = Paragraph::new("[Q] Quit  [S] Share Address  [M] Mute")
        .block(Block::default().borders(Borders::ALL));
    f.render_widget(footer, chunks[4]);
}
8. CLI Interface
Rust

// src/main.rs
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "radioghost")]
#[command(about = "Anonymous terminal radio. Broadcast. Vanish.")]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Start broadcasting
    Broadcast {
        /// Station name
        #[arg(short, long, default_value = "Ghost Radio")]
        name: String,

        /// Source: mic, file, or playlist
        #[arg(short, long, default_value = "mic")]
        source: String,

        /// File path (if source is file or playlist)
        #[arg(short, long)]
        path: Option<String>,
    },
    /// Listen to a station
    Listen {
        /// Onion address of the station
        address: String,
    },
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    match cli.command {
        Commands::Broadcast { name, source, path } => {
            run_broadcaster(name, source, path).await?;
        },
        Commands::Listen { address } => {
            run_listener(address).await?;
        }
    }
    Ok(())
}
9. Development Roadmap
text

Phase 1 (Weeks 1-3): Core Audio Pipeline
├── CPAL microphone capture
├── Opus encoding
├── Local HTTP audio server
└── Basic file playback

Phase 2 (Weeks 4-6): Tor Integration
├── Arti (pure Rust Tor) integration
├── Onion service creation
├── SOCKS5 listener client
└── Address sharing mechanism

Phase 3 (Weeks 7-9): TUI
├── Broadcaster UI (Ratatui)
├── Listener UI with waveform visualizer
├── Station directory (optional)
└── Playlist management

Phase 4 (Weeks 10-12): Polish
├── Playlist auto-crossfade
├── Multi-listener load testing
├── Package for Linux/Mac/Windows
└── Release
