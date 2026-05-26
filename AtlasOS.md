AtlasOS
A self-hosted, open-source alternative to Google Maps — for your entire personal world
🧭 Vision & Core Philosophy
AtlasOS is a fully local, offline-first, deeply personal mapping and place intelligence system. It is not a Google Maps clone. It is something far more personal and powerful.

AtlasOS lets you own your geographic memory. Every place you've ever been. Every restaurant you've loved. Every photo you've geotagged. Every place mentioned in your notes, emails, or browser history. Every business you've researched. Every trip you've planned. Every neighbourhood you're curious about — all pulled into a private, offline, queryable personal atlas.

Then it goes further: you can layer it with any public data source — crime stats, air quality, cost of living, coffee shop density, noise maps, political leanings — and build your own personally relevant map of the world, answering questions no commercial map tool will ever answer:

"Show me all restaurants I've visited and rated highly, on a map." "Find me a neighbourhood with low noise, walkable, <30min from city centre, and near a park." "Map everywhere I've been in the last 5 years from my photo GPS data." "Show me all places mentioned in my saved articles this month."

It uses OpenStreetMap data and completely local, offline tile rendering. Zero Google. Zero tracking. Forever.

🏗️ System Architecture
text

┌──────────────────────────────────────────────────────────────┐
│                       AtlasOS                                │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Data Ingestion Layer                    │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │    │
│  │  │  Photo   │ │ Browser  │ │  Notes / │ │  GPS   │  │    │
│  │  │  EXIF    │ │ History  │ │  Docs    │  │ Track  │  │    │
│  │  │  Parser  │ │ Geo-ext. │ │ Geo-NER  │ │ Import │  │    │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └───┬────┘  │    │
│  └───────│────────────│────────────│───────────│────────┘   │
│          └────────────┼────────────┘           │            │
│                       ▼                        ▼            │
│  ┌──────────────────────────────────────────────────────┐    │
│  │           Place Intelligence Engine                  │    │
│  │  (Geocoding via Nominatim local + OSM data)          │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Personal Atlas Database                 │    │
│  │  (SpatiaLite: spatial SQLite extension)              │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Layer Engine                            │    │
│  │  (Public data layers: OSM, air quality, cost etc.)   │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Tile Server (local offline)             │    │
│  │              (Martin tile server + OSM vector tiles) │    │
│  └──────────────────────────┬───────────────────────────┘    │
│                             ▼                               │
│  ┌──────────────────────────────────────────────────────┐    │
│  │              Map UI (MapLibre GL JS + Svelte)        │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
🗂️ Full Directory Structure
text

atlasos/
│
├── backend/                         # Go backend
│   ├── main.go
│   ├── config/
│   │   └── config.go
│   │
│   ├── api/
│   │   ├── router.go
│   │   └── handlers/
│   │       ├── places.go            # CRUD for personal places
│   │       ├── search.go            # Spatial + text search
│   │       ├── layers.go            # Data layer management
│   │       ├── ingest.go            # File ingestion endpoints
│   │       ├── query.go             # Natural language map queries
│   │       └── export.go            # Export as GeoJSON/KML/GPX
│   │
│   ├── ingestors/
│   │   ├── photo_exif.go            # Extract GPS from photo EXIF
│   │   ├── gpx.go                   # Import GPX tracks
│   │   ├── geojson.go               # Import GeoJSON
│   │   ├── browser_history.go       # Extract locations from browser history
│   │   └── text_geo.go              # NER: extract places from text/notes
│   │
│   ├── geocoder/
│   │   ├── nominatim.go             # Local Nominatim geocoding client
│   │   ├── cache.go                 # Cache geocoding results
│   │   └── reverse.go               # Reverse geocoding (coords → address)
│   │
│   ├── spatial/
│   │   ├── db.go                    # SpatiaLite connection
│   │   ├── index.go                 # R-tree spatial index
│   │   ├── search.go                # Bounding box + radius queries
│   │   └── query_builder.go         # Build spatial SQL from filter params
│   │
│   ├── layers/
│   │   ├── manager.go               # Layer management
│   │   ├── osm.go                   # OpenStreetMap POI layer
│   │   ├── air_quality.go           # OpenAQ data layer
│   │   └── custom.go                # User-uploaded GeoJSON layers
│   │
│   ├── nlquery/
│   │   ├── parser.go                # Parse natural language map queries
│   │   ├── intent.go                # Extract intent: find | show | compare | route
│   │   └── executor.go              # Execute parsed query on spatial DB
│   │
│   └── tileserver/
│       └── proxy.go                 # Proxy to local Martin tile server
│
├── frontend/                        # Svelte frontend
│   └── src/
│       ├── routes/
│       │   ├── +page.svelte         # Main map view
│       │   ├── places/+page.svelte  # Place library
│       │   ├── trips/+page.svelte   # Trip history
│       │   └── query/+page.svelte   # NL query interface
│       ├── components/
│       │   ├── Map.svelte           # MapLibre GL JS wrapper
│       │   ├── PlacePanel.svelte    # Place detail sidebar
│       │   ├── LayerControl.svelte  # Toggle data layers
│       │   ├── SearchBar.svelte     # Spatial + text search
│       │   ├── PhotoTimeline.svelte # Photo location timeline
│       │   ├── QueryBar.svelte      # Natural language query input
│       │   └── FilterPanel.svelte   # Attribute filters
│       └── lib/
│           ├── maplibre.ts          # MapLibre setup
│           └── api.ts
│
├── tiles/                           # Local OSM tile data (gitignored)
│   └── region.mbtiles               # Downloaded OSM MBTiles file
│
├── nominatim/                       # Local Nominatim instance (Docker)
│   └── docker-compose.yml
│
├── scripts/
│   ├── download_tiles.sh            # Download MBTiles for your region
│   ├── setup_nominatim.sh           # Set up local geocoder
│   └── import_photos.sh             # Batch import photo library
│
├── docker-compose.yml
├── Makefile
└── README.md
🗄️ Database Schema (SpatiaLite)
SQL

-- Enable SpatiaLite extension
SELECT InitSpatialMetaData();

-- Personal places (restaurants, homes, visited locations etc.)
CREATE TABLE places (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    category        TEXT,
    -- 'restaurant' | 'home' | 'work' | 'visited' | 'wishlist' | 'custom'
    description     TEXT,
    address         TEXT,
    source          TEXT,            -- How it was added: 'manual' | 'photo' | 'import'
    source_ref      TEXT,            -- Reference to source document/photo
    created_at      DATETIME NOT NULL,
    visited_at      DATETIME,
    user_rating     INTEGER,         -- 1-5 stars
    user_tags       TEXT,            -- JSON array
    user_notes      TEXT,
    is_favorite     BOOLEAN DEFAULT FALSE,
    metadata_json   TEXT
);

-- Add geometry column (SpatiaLite)
SELECT AddGeometryColumn('places', 'geom', 4326, 'POINT', 'XY');
SELECT CreateSpatialIndex('places', 'geom');

-- GPS tracks (from imported GPX files or phone tracking)
CREATE TABLE tracks (
    id              TEXT PRIMARY KEY,
    name            TEXT,
    source          TEXT,            -- 'gpx' | 'phone' | 'manual'
    started_at      DATETIME,
    ended_at        DATETIME,
    distance_km     REAL,
    point_count     INTEGER,
    metadata_json   TEXT
);

-- Add LineString geometry
SELECT AddGeometryColumn('tracks', 'geom', 4326, 'LINESTRING', 'XY');
SELECT CreateSpatialIndex('tracks', 'geom');

-- Individual track points
CREATE TABLE track_points (
    id              TEXT PRIMARY KEY,
    track_id        TEXT NOT NULL REFERENCES tracks(id),
    recorded_at     DATETIME NOT NULL,
    elevation_m     REAL,
    speed_kmh       REAL,
    accuracy_m      REAL
);
SELECT AddGeometryColumn('track_points', 'geom', 4326, 'POINT', 'XY');

-- Geotagged photos from your library
CREATE TABLE geotagged_photos (
    id              TEXT PRIMARY KEY,
    file_path       TEXT NOT NULL,
    filename        TEXT NOT NULL,
    taken_at        DATETIME,
    camera_model    TEXT,
    place_id        TEXT REFERENCES places(id),
    thumbnail_path  TEXT,
    exif_json       TEXT
);
SELECT AddGeometryColumn('geotagged_photos', 'geom', 4326, 'POINT', 'XY');
SELECT CreateSpatialIndex('geotagged_photos', 'geom');

-- User-defined data layers
CREATE TABLE layers (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    description     TEXT,
    layer_type      TEXT,   -- 'geojson' | 'osm' | 'air_quality' | 'custom'
    source_url      TEXT,
    is_visible      BOOLEAN DEFAULT TRUE,
    style_json      TEXT,   -- MapLibre layer style
    last_updated    DATETIME
);

-- Saved spatial queries / views
CREATE TABLE saved_queries (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    query_text      TEXT NOT NULL,   -- Natural language
    query_sql       TEXT,            -- Compiled SQL
    created_at      DATETIME NOT NULL,
    last_run        DATETIME
);
🗺️ Natural Language Query Engine
Go

// nlquery/parser.go

// Examples of natural language queries AtlasOS understands:

var QueryExamples = []string{
    // Personal memory queries
    "Show me all places I visited in Japan",
    "Where did I take the most photos in 2023?",
    "Find the restaurant I went to near the Eiffel Tower last spring",
    "Show my walking routes from last month",

    // Discovery queries
    "Find coffee shops within 500m of my home",
    "What's nearby my most visited location on weekdays?",
    "Show me all places I've marked as favourite",

    // Analysis queries
    "Which city have I spent the most time in?",
    "Show me a heatmap of everywhere I've been",
    "What's the furthest place I've ever visited from home?",

    // Research queries
    "Find neighbourhoods with low noise and good walkability",
    "Show areas in Berlin with good air quality",
    "Compare two neighbourhoods I'm considering moving to",
}

type ParsedQuery struct {
    Intent      string        // "find" | "show" | "analyze" | "compare" | "route"
    Filters     []SpatialFilter
    Temporal    *TimeRange
    Subject     string        // What we're looking for
    Context     string        // Contextual anchor (near home, in Japan, etc.)
    Limit       int
}
🖥️ Map UI Features
text

┌─────────────────────── AtlasOS ──────────────────────────────┐
│  🔍 [Find coffee shops I liked near my home_____________] [⏎]│
│  [Layers ▼] [My Places] [Photos] [Tracks] [Air Quality]      │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│           ╔═══════════════════════════════════╗              │
│           ║   Interactive Map (MapLibre GL)   ║              │
│           ║                                   ║              │
│           ║   📍 My Home                      ║              │
│           ║   ☕ Brew Lab [★★★★★] 320m        ║              │
│           ║   ☕ Monmouth [★★★★☆] 450m        ║              │
│           ║   📸 47 photos nearby             ║              │
│           ║   🛤️ 12 walks through here        ║              │
│           ║                                   ║              │
│           ╚═══════════════════════════════════╝              │
│                                                              │
│  PLACE DETAIL: Brew Lab                                      │
│  ────────────────────                                        │
│  ★★★★★  Visited 7 times  First: Mar 2022                    │
│  Your note: "Best flat white in the city. Window seat."      │
│  Photos here: 📸 📸 📸                                       │
│  Tags: coffee, work, favourite                               │
└──────────────────────────────────────────────────────────────┘
📦 Tech Stack Summary
Layer	Technology
Backend	Go 1.22
Spatial Database	SQLite + SpatiaLite
Geocoding	Nominatim (local Docker instance)
Map Tiles	OSM MBTiles + Martin tile server
Map Rendering	MapLibre GL JS
Natural Language Query	Go + local Ollama (for NL parsing)
Photo EXIF	exifgo
GPX Parsing	Custom Go parser
Frontend	Svelte 5 + TailwindCSS
Containerization	Docker + docker-compose
🚀 Build Phases
text

Phase 1 — Core Atlas (3 weeks)
├── SpatiaLite setup + schema
├── Local Nominatim geocoder
├── Martin tile server + OSM tiles
├── Basic MapLibre map UI
└── Manual place CRUD

Phase 2 — Data Ingestion (3 weeks)
├── Photo EXIF GPS extractor
├── GPX track importer
├── Browser history geo-extractor
└── Text/notes geo-NER

Phase 3 — NL Query Engine (2 weeks)
├── Query intent parser
├── Spatial SQL builder
├── LLM-enhanced NL parsing
└── Query result visualization

Phase 4 — Layers + Analysis (2 weeks)
├── OSM POI layer
├── Air quality layer
├── Heatmap generator
└── Trip statistics

Phase 5 — Polish + Mobile (3 weeks)
├── Mobile-responsive UI
├── Offline tile caching
├── Export (GeoJSON, KML, GPX)
└── PWA for mobile use
