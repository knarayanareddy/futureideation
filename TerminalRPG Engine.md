🎲 TerminalRPG Engine
Overview
A fully open-source, Lua-scriptable, multiplayer terminal RPG engine where players connect via SSH. Worlds are procedurally generated from a seed word. Built on Rust + Ratatui + russh.

Architecture Overview
text

┌────────────────────────────────────────────────────────────────┐
│                     TerminalRPG Engine                          │
├──────────────┬───────────────────┬──────────────────────────────┤
│  SSH Server  │   Game Engine     │   Scripting & World Gen      │
│  (russh,     │  (ECS, tick loop, │  (Lua API, procedural        │
│   multi-     │   combat, items)  │   world from seed)           │
│   player)    │                   │                              │
└──────────────┴───────────────────┴──────────────────────────────┘
Full Component Breakdown
1. Project Structure
text

termrpg/
├── src/
│   ├── main.rs
│   ├── server/
│   │   ├── mod.rs
│   │   ├── ssh_server.rs       # SSH connection handler (russh)
│   │   ├── session.rs          # Per-player session
│   │   └── auth.rs             # Key-based auth
│   ├── engine/
│   │   ├── mod.rs
│   │   ├── world.rs            # World state
│   │   ├── map.rs              # Map tiles and chunks
│   │   ├── entity.rs           # ECS entities
│   │   ├── components.rs       # ECS components
│   │   ├── systems.rs          # ECS systems
│   │   ├── combat.rs           # Combat system
│   │   ├── items.rs            # Item system
│   │   ├── quests.rs           # Quest system
│   │   └── events.rs           # Event bus
│   ├── worldgen/
│   │   ├── mod.rs
│   │   ├── generator.rs        # Seed-based world generation
│   │   ├── biomes.rs           # Biome definitions
│   │   ├── dungeons.rs         # Dungeon generation
│   │   └── npcs.rs             # NPC placement
│   ├── render/
│   │   ├── mod.rs
│   │   ├── renderer.rs         # Ratatui renderer
│   │   ├── viewport.rs         # Camera/viewport
│   │   └── widgets.rs          # Custom UI widgets
│   ├── lua/
│   │   ├── mod.rs
│   │   ├── api.rs              # Lua API bindings
│   │   └── sandbox.rs          # Safe Lua sandbox
│   └── db/
│       ├── mod.rs
│       └── world_db.rs         # Sled persistent storage
├── scripts/                    # Built-in Lua scripts
│   ├── world/
│   │   └── starter_village.lua
│   ├── quests/
│   │   └── tutorial.lua
│   └── npcs/
│       └── innkeeper.lua
├── Cargo.toml
└── README.md
2. Cargo.toml
toml

[package]
name = "termrpg"
version = "0.1.0"
edition = "2021"

[dependencies]
# TUI Rendering
ratatui = "0.26"
crossterm = "0.27"

# SSH Server
russh = "0.44"
russh-keys = "0.44"

# ECS
hecs = "0.10"                  # Lightweight ECS

# World Generation
noise = "0.9"                  # Perlin/Simplex noise
rand = { version = "0.8", features = ["getrandom"] }
rand_chacha = "0.3"            # Deterministic RNG from seed

# Lua Scripting
mlua = { version = "0.9", features = ["lua54", "vendored"] }

# Storage
sled = "0.34"
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Async
tokio = { version = "1", features = ["full"] }
async-trait = "0.1"

# Utilities
anyhow = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
uuid = { version = "1", features = ["v4"] }
hashbrown = "0.14"
3. SSH Server (russh)
Rust

// src/server/ssh_server.rs
use russh::*;
use russh::server::*;
use russh_keys::*;
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{Mutex, RwLock};
use crate::engine::World;
use super::session::PlayerSession;

pub struct GameServer {
    world: Arc<RwLock<World>>,
    sessions: Arc<Mutex<HashMap<String, Arc<Mutex<PlayerSession>>>>>,
}

impl GameServer {
    pub fn new(world: Arc<RwLock<World>>) -> Self {
        GameServer {
            world,
            sessions: Arc::new(Mutex::new(HashMap::new())),
        }
    }

    pub async fn run(&self, host: &str, port: u16) -> anyhow::Result<()> {
        let config = Arc::new(Config {
            inactivity_timeout: Some(std::time::Duration::from_secs(3600)),
            auth_rejection_time: std::time::Duration::from_secs(1),
            auth_rejection_time_initial: Some(std::time::Duration::from_secs(0)),
            keys: vec![
                key::KeyPair::generate_ed25519().unwrap()
            ],
            ..Default::default()
        });

        let server = self.clone();
        let addr = format!("{}:{}", host, port);
        println!("⚔️  TerminalRPG Server listening on {}", addr);

        russh::server::run(config, &addr, server).await?;
        Ok(())
    }
}

struct GameHandler {
    player_id: String,
    world: Arc<RwLock<World>>,
    session: Arc<Mutex<PlayerSession>>,
    handle: Option<Handle>,
}

#[async_trait::async_trait]
impl Handler for GameHandler {
    type Error = anyhow::Error;

    async fn channel_open_session(
        &mut self,
        channel: Channel<Msg>,
        _session: &mut Session,
    ) -> Result<bool, Self::Error> {
        Ok(true)
    }

    async fn auth_publickey(
        &mut self,
        user: &str,
        _public_key: &key::PublicKey,
    ) -> Result<Auth, Self::Error> {
        // Accept all keys — username becomes player name
        self.player_id = user.to_string();
        Ok(Auth::Accept)
    }

    async fn data(
        &mut self,
        channel: ChannelId,
        data: &[u8],
        _session: &mut Session,
    ) -> Result<(), Self::Error> {
        // Handle player input
        let mut session = self.session.lock().await;
        session.handle_input(data).await;
        Ok(())
    }

    async fn pty_request(
        &mut self,
        _channel: ChannelId,
        _term: &str,
        col_width: u32,
        row_height: u32,
        _pix_width: u32,
        _pix_height: u32,
        _modes: &[(Pty, u32)],
        _session: &mut Session,
    ) -> Result<bool, Self::Error> {
        let mut session = self.session.lock().await;
        session.set_terminal_size(col_width as u16, row_height as u16);
        Ok(true)
    }

    async fn window_change_request(
        &mut self,
        _channel: ChannelId,
        col_width: u32,
        row_height: u32,
        _pix_width: u32,
        _pix_height: u32,
        _session: &mut Session,
    ) -> Result<(), Self::Error> {
        let mut session = self.session.lock().await;
        session.set_terminal_size(col_width as u16, row_height as u16);
        Ok(())
    }
}
4. World Generation (Seed-Based)
Rust

// src/worldgen/generator.rs
use noise::{NoiseFn, Perlin, Seedable};
use rand::{SeedableRng, Rng};
use rand_chacha::ChaCha20Rng;
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};
use crate::engine::{Tile, TileType, World, Chunk};

pub struct WorldGenerator {
    seed: u64,
    rng: ChaCha20Rng,
    elevation_noise: Perlin,
    moisture_noise: Perlin,
    temperature_noise: Perlin,
}

impl WorldGenerator {
    pub fn from_word(seed_word: &str) -> Self {
        // Convert word to deterministic u64 seed
        let mut hasher = DefaultHasher::new();
        seed_word.hash(&mut hasher);
        let seed = hasher.finish();

        let rng = ChaCha20Rng::seed_from_u64(seed);

        WorldGenerator {
            seed,
            rng,
            elevation_noise: Perlin::new(seed as u32),
            moisture_noise: Perlin::new((seed >> 16) as u32),
            temperature_noise: Perlin::new((seed >> 32) as u32),
        }
    }

    pub fn generate_chunk(&mut self, chunk_x: i32, chunk_y: i32) -> Chunk {
        const CHUNK_SIZE: usize = 32;
        let mut tiles = vec![vec![Tile::default(); CHUNK_SIZE]; CHUNK_SIZE];

        for y in 0..CHUNK_SIZE {
            for x in 0..CHUNK_SIZE {
                let world_x = (chunk_x * CHUNK_SIZE as i32 + x as i32) as f64;
                let world_y = (chunk_y * CHUNK_SIZE as i32 + y as i32) as f64;

                let elevation = self.elevation_noise.get([
                    world_x * 0.01,
                    world_y * 0.01,
                ]);
                let moisture = self.moisture_noise.get([
                    world_x * 0.008 + 100.0,
                    world_y * 0.008 + 100.0,
                ]);
                let temperature = self.temperature_noise.get([
                    world_x * 0.006 + 200.0,
                    world_y * 0.006 + 200.0,
                ]);

                tiles[y][x] = self.elevation_to_tile(elevation, moisture, temperature);
            }
        }

        // Place structures
        self.place_structures(&mut tiles, chunk_x, chunk_y);

        Chunk {
            tiles,
            chunk_x,
            chunk_y,
        }
    }

    fn elevation_to_tile(&self, elevation: f64, moisture: f64, temp: f64) -> Tile {
        let tile_type = if elevation < -0.3 {
            TileType::DeepWater
        } else if elevation < -0.1 {
            TileType::ShallowWater
        } else if elevation < 0.0 {
            TileType::Sand
        } else if elevation < 0.3 {
            if moisture > 0.3 {
                TileType::Forest
            } else {
                TileType::Grass
            }
        } else if elevation < 0.5 {
            if temp < -0.2 {
                TileType::Snow
            } else {
                TileType::Hills
            }
        } else {
            TileType::Mountain
        };

        Tile {
            tile_type,
            passable: !matches!(tile_type,
                TileType::DeepWater | TileType::Mountain),
            glyph: tile_type.glyph(),
            color: tile_type.color(),
        }
    }

    fn place_structures(&mut self, tiles: &mut Vec<Vec<Tile>>, cx: i32, cy: i32) {
        // Place villages every ~10 chunks
        if (cx % 10 == 0) && (cy % 10 == 0) && self.rng.gen_bool(0.3) {
            self.place_village(tiles);
        }

        // Place dungeons scattered around
        if self.rng.gen_bool(0.05) {
            self.place_dungeon_entrance(tiles);
        }
    }

    fn place_village(&mut self, tiles: &mut Vec<Vec<Tile>>) {
        let center_x = 10 + self.rng.gen_range(0..12) as usize;
        let center_y = 10 + self.rng.gen_range(0..12) as usize;

        // Inn
        tiles[center_y][center_x].tile_type = TileType::Inn;
        tiles[center_y][center_x].glyph = 'I';

        // Market
        tiles[center_y + 2][center_x].tile_type = TileType::Market;
        tiles[center_y + 2][center_x].glyph = 'M';

        // Houses
        for i in 0..4 {
            let hx = center_x + (i % 2) * 2;
            let hy = center_y + 4 + (i / 2) * 2;
            if hx < 32 && hy < 32 {
                tiles[hy][hx].tile_type = TileType::House;
                tiles[hy][hx].glyph = '#';
            }
        }
    }

    fn place_dungeon_entrance(&mut self, tiles: &mut Vec<Vec<Tile>>) {
        let x = self.rng.gen_range(5..27) as usize;
        let y = self.rng.gen_range(5..27) as usize;
        tiles[y][x].tile_type = TileType::DungeonEntrance;
        tiles[y][x].glyph = '>';
        tiles[y][x].passable = true;
    }
}
5. Tile Types & Glyphs
Rust

// src/engine/map.rs
use ratatui::style::Color;

#[derive(Debug, Clone, Copy, PartialEq, Default)]
pub enum TileType {
    #[default]
    Grass,
    Forest,
    DeepWater,
    ShallowWater,
    Sand,
    Hills,
    Mountain,
    Snow,
    Inn,
    Market,
    House,
    DungeonEntrance,
    DungeonFloor,
    DungeonWall,
    Path,
}

impl TileType {
    pub fn glyph(&self) -> char {
        match self {
            TileType::Grass          => '.',
            TileType::Forest         => '♣',
            TileType::DeepWater      => '~',
            TileType::ShallowWater   => '≈',
            TileType::Sand           => ':',
            TileType::Hills          => '∩',
            TileType::Mountain       => '▲',
            TileType::Snow           => '*',
            TileType::Inn            => 'I',
            TileType::Market         => 'M',
            TileType::House          => '⌂',
            TileType::DungeonEntrance => '>',
            TileType::DungeonFloor   => '·',
            TileType::DungeonWall    => '█',
            TileType::Path           => '─',
        }
    }

    pub fn color(&self) -> Color {
        match self {
            TileType::Grass          => Color::Green,
            TileType::Forest         => Color::DarkGray,
            TileType::DeepWater      => Color::Blue,
            TileType::ShallowWater   => Color::Cyan,
            TileType::Sand           => Color::Yellow,
            TileType::Hills          => Color::Rgb(139, 90, 43),
            TileType::Mountain       => Color::White,
            TileType::Snow           => Color::White,
            TileType::Inn            => Color::Rgb(200, 150, 50),
            TileType::Market         => Color::Rgb(200, 100, 50),
            TileType::House          => Color::Rgb(180, 120, 60),
            TileType::DungeonEntrance => Color::Magenta,
            TileType::DungeonFloor   => Color::DarkGray,
            TileType::DungeonWall    => Color::Gray,
            TileType::Path           => Color::Rgb(150, 120, 80),
        }
    }
}

#[derive(Debug, Clone, Default)]
pub struct Tile {
    pub tile_type: TileType,
    pub passable: bool,
    pub glyph: char,
    pub color: Color,
}
6. ECS Components
Rust

// src/engine/components.rs
use serde::{Deserialize, Serialize};
use ratatui::style::Color;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Position {
    pub x: i32,
    pub y: i32,
    pub chunk_x: i32,
    pub chunk_y: i32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Renderable {
    pub glyph: char,
    pub color: Color,
    pub bg_color: Option<Color>,
    pub layer: u8,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Stats {
    pub hp: i32,
    pub max_hp: i32,
    pub mp: i32,
    pub max_mp: i32,
    pub attack: i32,
    pub defense: i32,
    pub speed: i32,
    pub level: u32,
    pub exp: u32,
    pub exp_to_next: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct PlayerTag {
    pub player_id: String,
    pub username: String,
    pub class: PlayerClass,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum PlayerClass {
    Warrior,
    Mage,
    Rogue,
    Healer,
    Bard,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Inventory {
    pub items: Vec<Item>,
    pub gold: u32,
    pub max_slots: usize,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Item {
    pub id: String,
    pub name: String,
    pub item_type: ItemType,
    pub stats_bonus: Option<Stats>,
    pub value: u32,
    pub stackable: bool,
    pub count: u32,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ItemType {
    Weapon,
    Armor,
    Potion,
    Quest,
    Key,
    Food,
    Scroll,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct NPC {
    pub name: String,
    pub dialog_script: String,  // Lua script file
    pub faction: String,
    pub hostile: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Combat {
    pub target: Option<hecs::Entity>,
    pub in_combat: bool,
    pub action_points: i32,
}

#[derive(Debug, Clone)]
pub struct QuestLog {
    pub active_quests: Vec<Quest>,
    pub completed_quests: Vec<String>,
}

#[derive(Debug, Clone)]
pub struct Quest {
    pub id: String,
    pub name: String,
    pub description: String,
    pub objectives: Vec<QuestObjective>,
    pub reward_gold: u32,
    pub reward_exp: u32,
}

#[derive(Debug, Clone)]
pub struct QuestObjective {
    pub description: String,
    pub completed: bool,
    pub current: u32,
    pub required: u32,
}
7. Renderer (Ratatui)
Rust

// src/render/renderer.rs
use ratatui::{
    layout::{Constraint, Direction, Layout, Rect},
    style::{Color, Modifier, Style},
    text::{Line, Span},
    widgets::{Block, Borders, Gauge, List, ListItem, Paragraph, Clear},
    Frame,
};
use crate::engine::{World, components::*};
use crate::server::session::PlayerSession;

pub struct GameRenderer;

impl GameRenderer {
    pub fn render(f: &mut Frame, world: &World, session: &PlayerSession) {
        let chunks = Layout::default()
            .direction(Direction::Vertical)
            .constraints([
                Constraint::Length(1),   // Title bar
                Constraint::Min(0),      // Main game area
                Constraint::Length(5),   // Message log
                Constraint::Length(3),   // Status bar
            ])
            .split(f.size());

        let main_chunks = Layout::default()
            .direction(Direction::Horizontal)
            .constraints([
                Constraint::Min(0),      // Map viewport
                Constraint::Length(22),  // Sidebar
            ])
            .split(chunks[1]);

        Self::render_titlebar(f, chunks[0], session);
        Self::render_map(f, main_chunks[0], world, session);
        Self::render_sidebar(f, main_chunks[1], world, session);
        Self::render_messages(f, chunks[2], session);
        Self::render_status(f, chunks[3], world, session);
    }

    fn render_titlebar(f: &mut Frame, area: Rect, session: &PlayerSession) {
        let title = Paragraph::new(Line::from(vec![
            Span::styled("⚔️  TerminalRPG ", Style::default()
                .fg(Color::Yellow)
                .add_modifier(Modifier::BOLD)),
            Span::raw("│ "),
            Span::styled(&session.player_name, Style::default().fg(Color::Cyan)),
            Span::raw(" │ "),
            Span::styled(
                format!("World: {}", session.world_seed),
                Style::default().fg(Color::DarkGray)
            ),
        ]));
        f.render_widget(title, area);
    }

    fn render_map(f: &mut Frame, area: Rect, world: &World, session: &PlayerSession) {
        let viewport = &session.viewport;
        let mut lines: Vec<Line> = Vec::new();

        let visible_height = area.height as i32;
        let visible_width = (area.width / 2) as i32; // 2 chars per cell

        for dy in 0..visible_height {
            let mut spans = Vec::new();
            for dx in 0..visible_width {
                let world_x = viewport.x + dx;
                let world_y = viewport.y + dy;

                // Check for entities at this position first
                if let Some(entity_render) = world.get_entity_at(world_x, world_y) {
                    spans.push(Span::styled(
                        format!("{} ", entity_render.glyph),
                        Style::default().fg(entity_render.color)
                    ));
                } else if let Some(tile) = world.get_tile(world_x, world_y) {
                    spans.push(Span::styled(
                        format!("{} ", tile.glyph),
                        Style::default().fg(tile.color)
                    ));
                } else {
                    spans.push(Span::raw("  ")); // Ungenerated chunk
                }
            }
            lines.push(Line::from(spans));
        }

        let map_widget = Paragraph::new(lines)
            .block(Block::default().borders(Borders::ALL).title("World"));
        f.render_widget(map_widget, area);
    }

    fn render_sidebar(f: &mut Frame, area: Rect, world: &World, session: &PlayerSession) {
        let sidebar_chunks = Layout::default()
            .direction(Direction::Vertical)
            .constraints([
                Constraint::Length(10),  // Player stats
                Constraint::Length(8),   // Equipment
                Constraint::Min(0),      // Nearby players
            ])
            .split(area);

        // Player stats
        if let Some(stats) = &session.player_stats {
            let hp_pct = (stats.hp as f64 / stats.max_hp as f64 * 100.0) as u16;
            let mp_pct = (stats.mp as f64 / stats.max_mp as f64 * 100.0) as u16;

            let stats_text = vec![
                Line::from(format!("⚔️  ATK: {}   🛡️  DEF: {}", stats.attack, stats.defense)),
                Line::from(format!("⚡ SPD: {}   💫 LVL: {}", stats.speed, stats.level)),
                Line::from(format!("✨ EXP: {}/{}", stats.exp, stats.exp_to_next)),
            ];

            let stats_block = Paragraph::new(stats_text)
                .block(Block::default().title("Stats").borders(Borders::ALL));
            f.render_widget(stats_block, sidebar_chunks[0]);
        }

        // Nearby players
        let nearby: Vec<ListItem> = world
            .get_nearby_players(&session.player_id, 20)
            .iter()
            .map(|p| ListItem::new(format!("@{}", p)))
            .collect();

        let nearby_widget = List::new(nearby)
            .block(Block::default().title("Nearby").borders(Borders::ALL));
        f.render_widget(nearby_widget, sidebar_chunks[2]);
    }

    fn render_messages(f: &mut Frame, area: Rect, session: &PlayerSession) {
        let messages: Vec<Line> = session.message_log
            .iter()
            .rev()
            .take(area.height as usize - 2)
            .rev()
            .map(|m| Line::from(Span::styled(m.text.as_str(),
                Style::default().fg(m.color))))
            .collect();

        let log = Paragraph::new(messages)
            .block(Block::default().title("Messages").borders(Borders::ALL));
        f.render_widget(log, area);
    }

    fn render_status(f: &mut Frame, area: Rect, world: &World, session: &PlayerSession) {
        let controls = Paragraph::new(
            "[WASD/←↑↓→] Move  [I] Inventory  [Q] Quests  [T] Talk  [A] Attack  [/] Chat"
        ).block(Block::default().borders(Borders::ALL));
        f.render_widget(controls, area);
    }
}
8. Lua Scripting API
Rust

// src/lua/api.rs
use mlua::prelude::*;
use std::sync::Arc;
use tokio::sync::RwLock;
use crate::engine::World;

pub fn create_lua_env(world: Arc<RwLock<World>>) -> LuaResult<Lua> {
    let lua = Lua::new();
    let globals = lua.globals();

    // World API
    let world_clone = world.clone();
    let get_tile = lua.create_function(move |_, (x, y): (i32, i32)| {
        let world = world_clone.blocking_read();
        if let Some(tile) = world.get_tile(x, y) {
            Ok(Some(tile.glyph.to_string()))
        } else {
            Ok(None)
        }
    })?;
    globals.set("get_tile", get_tile)?;

    // Spawn NPC
    let world_clone = world.clone();
    let spawn_npc = lua.create_function(move |_, (name, x, y, script): (String, i32, i32, String)| {
        let mut world = world_clone.blocking_write();
        world.spawn_npc(name, x, y, script);
        Ok(())
    })?;
    globals.set("spawn_npc", spawn_npc)?;

    // Send message to player
    let world_clone = world.clone();
    let send_message = lua.create_function(move |_, (player_id, msg): (String, String)| {
        let world = world_clone.blocking_read();
        world.send_message(&player_id, &msg, ratatui::style::Color::White);
        Ok(())
    })?;
    globals.set("send_message", send_message)?;

    // Give item to player
    let world_clone = world.clone();
    let give_item = lua.create_function(move |_, (player_id, item_id): (String, String)| {
        let mut world = world_clone.blocking_write();
        world.give_item(&player_id, &item_id);
        Ok(())
    })?;
    globals.set("give_item", give_item)?;

    // Play sound effect (terminal bell or ASCII animation)
    globals.set("play_sound", lua.create_function(|_, sound: String| {
        match sound.as_str() {
            "bell" => print!("\x07"),
            _ => {}
        }
        Ok(())
    })?)?;

    Ok(lua)
}
Lua

-- scripts/npcs/innkeeper.lua
-- Example NPC dialog script

local innkeeper = {}

function innkeeper.greet(player_id, player_name)
    send_message(player_id, "Innkeeper: \"Welcome to the Rusty Anchor, " .. player_name .. "!\"")
    send_message(player_id, "Innkeeper: \"Would you like a room? [1] Yes (5g)  [2] No\"")
end

function innkeeper.on_choice(player_id, choice)
    if choice == "1" then
        if get_player_gold(player_id) >= 5 then
            take_gold(player_id, 5)
            heal_player(player_id, 100)
            send_message(player_id, "You rest and recover all HP. 💤")
            send_message(player_id, "Innkeeper: \"Sweet dreams!\"")
        else
            send_message(player_id, "Innkeeper: \"Sorry, you don't have enough gold.\"")
        end
    else
        send_message(player_id, "Innkeeper: \"Come back anytime!\"")
    end
end

return innkeeper
Lua

-- scripts/quests/tutorial.lua
local quest = {}

quest.id = "tutorial_01"
quest.name = "First Steps"
quest.description = "Find the old wizard in the village square."

function quest.on_start(player_id)
    send_message(player_id, "📜 New Quest: First Steps")
    send_message(player_id, "   Find the old wizard in the village square.")
    spawn_npc("Old Wizard", 15, 15, "scripts/npcs/wizard.lua")
end

function quest.on_complete(player_id)
    give_item(player_id, "sword_iron")
    give_item(player_id, "potion_health_small")
    send_message(player_id, "✅ Quest Complete: First Steps!")
    send_message(player_id, "   Rewards: Iron Sword, Health Potion")
end

return quest
9. Development Roadmap
text

Phase 1 (Weeks 1-4): Core Engine
├── ECS setup (hecs)
├── SSH server (russh) + player sessions
├── Basic map rendering (Ratatui)
└── Player movement + viewport

Phase 2 (Weeks 5-8): World Generation
├── Seed-to-world generation (Perlin noise)
├── Biome system (grass, forest, water, mountain)
├── Village + dungeon placement
└── Chunk-based loading

Phase 3 (Weeks 9-12): Gameplay Systems
├── Combat system
├── Inventory + items
├── NPC system + Lua dialog scripting
└── Quest system

Phase 4 (Weeks 13-16): Multiplayer & Polish
├── Multi-player sync (shared world state)
├── In-game chat
├── Lua modding API documentation
└── Public server + release
