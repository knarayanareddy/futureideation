SoundDNA
Composes original music from your taste, locally
🧭 Vision & Core Philosophy
SoundDNA is a 100% local, offline-first music AI that does three things:

Analyzes your existing music library to build a deep audio fingerprint of your taste
Generates brand new, original music that matches your fingerprint
Evolves that fingerprint over time as your taste changes
No cloud. No subscription. No data leaves your machine. It runs on your GPU (or CPU if no GPU). It's the anti-Spotify AI DJ — because your taste is yours, and the music it makes is yours too.

🏗️ System Architecture
text

┌──────────────────────────────────────────────────────────────┐
│                      SoundDNA System                         │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Music Library Scanner                  │     │
│  │  (Scans local folders, reads audio files)           │     │
│  └──────────────────┬──────────────────────────────────┘     │
│                     │                                        │
│                     ▼                                        │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Audio Feature Extractor                │     │
│  │  (librosa: tempo, key, mood, timbre, spectral       │     │
│  │   features, chroma, MFCC, onset strength)           │     │
│  └──────────────────┬──────────────────────────────────┘     │
│                     │                                        │
│                     ▼                                        │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Taste Profile Engine                   │     │
│  │  (Clusters features → builds TasteVector)           │     │
│  │  (UMAP dimensionality reduction + K-means)          │     │
│  └──────────────────┬──────────────────────────────────┘     │
│                     │                                        │
│                     ▼                                        │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Generation Engine                      │     │
│  │  (Facebook MusicGen — local, quantized)             │     │
│  │  Conditioned on TasteVector + user prompt           │     │
│  └──────────────────┬──────────────────────────────────┘     │
│                     │                                        │
│                     ▼                                        │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              Post-Processing Pipeline               │     │
│  │  (Normalization, EQ matching, fade in/out)          │     │
│  └──────────────────┬──────────────────────────────────┘     │
│                     │                                        │
│                     ▼                                        │
│  ┌─────────────────────────────────────────────────────┐     │
│  │              SQLite Library                         │     │
│  │  (Stores fingerprints, generated tracks,            │     │
│  │   evolution history, user ratings)                  │     │
│  └─────────────────────────────────────────────────────┘     │
│                     │                                        │
│                     ▼                                        │
│  ┌─────────────────────────────────────────────────────┐     │
│  │         Terminal UI (Textual) or Web UI (Svelte)    │     │
│  └─────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
🗂️ Full Directory Structure
text

sounddna/
│
├── sounddna/                        # Main Python package
│   │
│   ├── __init__.py
│   ├── main.py                      # CLI entry point
│   ├── config.py                    # Config dataclass (paths, model, GPU)
│   │
│   ├── scanner/
│   │   ├── __init__.py
│   │   ├── library_scanner.py       # Walk directories, find audio files
│   │   ├── file_validator.py        # Validate audio files, check encoding
│   │   ├── metadata_reader.py       # Read ID3/Vorbis tags (mutagen)
│   │   └── watch_daemon.py          # inotify/FSEvents watcher for new files
│   │
│   ├── extractor/
│   │   ├── __init__.py
│   │   ├── feature_extractor.py     # Main extraction orchestrator
│   │   ├── temporal.py              # Tempo, BPM, beat grid
│   │   ├── harmonic.py              # Key, chord progression, chroma features
│   │   ├── timbral.py               # MFCC, spectral centroid, rolloff, flux
│   │   ├── rhythmic.py              # Onset strength, rhythm patterns
│   │   ├── energy.py                # RMS energy, dynamic range, loudness
│   │   ├── mood.py                  # Valence/arousal estimation
│   │   └── cache.py                 # Cache extracted features (joblib)
│   │
│   ├── profile/
│   │   ├── __init__.py
│   │   ├── taste_vector.py          # TasteVector dataclass + serialization
│   │   ├── clusterer.py             # K-means clustering of feature space
│   │   ├── reducer.py               # UMAP dimensionality reduction
│   │   ├── builder.py               # Builds TasteVector from cluster centroids
│   │   ├── evolution.py             # Tracks how taste changes over time
│   │   └── comparator.py            # Compare two TasteVectors
│   │
│   ├── generator/
│   │   ├── __init__.py
│   │   ├── engine.py                # Main generation orchestrator
│   │   ├── prompt_builder.py        # TasteVector → MusicGen text prompt
│   │   ├── musicgen_wrapper.py      # Local MusicGen inference wrapper
│   │   ├── conditioner.py           # Audio conditioning (melody continuation)
│   │   ├── session.py               # Generation session (multi-track)
│   │   └── playlist.py              # Auto-playlist generation
│   │
│   ├── postprocess/
│   │   ├── __init__.py
│   │   ├── normalizer.py            # Loudness normalization (pyloudnorm)
│   │   ├── eq_matcher.py            # Match EQ curve to library average
│   │   ├── fader.py                 # Fade in/out, crossfade
│   │   └── exporter.py              # Export to WAV/FLAC/MP3
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── database.py              # SQLite connection + init
│   │   ├── migrations.py            # Schema migrations
│   │   └── queries/
│   │       ├── tracks.py
│   │       ├── features.py
│   │       ├── profiles.py
│   │       └── generated.py
│   │
│   ├── ui/
│   │   ├── tui/
│   │   │   ├── app.py               # Textual TUI app
│   │   │   ├── screens/
│   │   │   │   ├── home.py          # Dashboard
│   │   │   │   ├── library.py       # Library browser
│   │   │   │   ├── profile.py       # Taste profile visualizer
│   │   │   │   ├── generate.py      # Generation controls
│   │   │   │   └── player.py        # Built-in audio player
│   │   │   └── widgets/
│   │   │       ├── taste_radar.py   # Radar chart of taste dimensions
│   │   │       ├── waveform.py      # ASCII waveform display
│   │   │       ├── progress_bar.py
│   │   │       └── track_list.py
│   │   └── web/                     # Optional Svelte web UI
│   │       ├── src/
│   │       └── package.json
│   │
│   └── api/
│       ├── __init__.py
│       ├── server.py                # FastAPI server (for web UI / CLI bridge)
│       └── routes/
│           ├── library.py
│           ├── profile.py
│           ├── generate.py
│           └── player.py
│
├── models/                          # Downloaded model weights (gitignored)
│   └── musicgen-small/              # or musicgen-medium
│
├── scripts/
│   ├── download_models.sh           # One-shot model download script
│   ├── benchmark_gpu.py             # Check if GPU is suitable
│   └── migrate_db.py
│
├── tests/
│   ├── test_extractor.py
│   ├── test_profile.py
│   ├── test_generator.py
│   └── fixtures/                    # Small test audio files
│
├── pyproject.toml
├── requirements.txt
├── requirements-gpu.txt             # CUDA-specific deps
├── Makefile
├── Dockerfile
└── README.md
🗄️ Database Schema
SQL

-- Tracks in your library
CREATE TABLE tracks (
    id              TEXT PRIMARY KEY,       -- UUID
    file_path       TEXT UNIQUE NOT NULL,
    filename        TEXT NOT NULL,
    title           TEXT,
    artist          TEXT,
    album           TEXT,
    genre           TEXT,
    duration_sec    REAL,
    sample_rate     INTEGER,
    channels        INTEGER,
    file_size_bytes INTEGER,
    date_added      DATETIME NOT NULL,
    last_modified   DATETIME,
    is_indexed      BOOLEAN DEFAULT FALSE,
    index_error     TEXT                   -- NULL if indexed successfully
);

-- Extracted audio features per track
CREATE TABLE features (
    id              TEXT PRIMARY KEY,
    track_id        TEXT NOT NULL REFERENCES tracks(id),
    extracted_at    DATETIME NOT NULL,
    -- Temporal
    bpm             REAL,
    beat_regularity REAL,
    onset_density   REAL,
    -- Harmonic
    key_root        TEXT,                  -- e.g. "C", "F#"
    key_mode        TEXT,                  -- "major" | "minor"
    key_confidence  REAL,
    chroma_vector   TEXT,                  -- JSON: 12-dim chroma
    -- Timbral
    mfcc_mean       TEXT,                  -- JSON: 20-dim MFCC mean
    mfcc_std        TEXT,                  -- JSON: 20-dim MFCC std
    spectral_centroid REAL,
    spectral_rolloff  REAL,
    spectral_flux     REAL,
    zero_crossing_rate REAL,
    -- Energy
    rms_mean        REAL,
    rms_std         REAL,
    dynamic_range   REAL,
    loudness_lufs   REAL,
    -- Mood (estimated)
    valence         REAL,                  -- 0.0 (negative) → 1.0 (positive)
    arousal         REAL,                  -- 0.0 (calm) → 1.0 (energetic)
    -- Reduced embedding
    umap_x          REAL,
    umap_y          REAL,
    umap_z          REAL
);

-- User's taste profiles (snapshots over time)
CREATE TABLE taste_profiles (
    id              TEXT PRIMARY KEY,
    created_at      DATETIME NOT NULL,
    track_count     INTEGER,
    -- Aggregate taste dimensions (all 0.0-1.0)
    avg_bpm         REAL,
    bpm_variance    REAL,
    preferred_keys  TEXT,                  -- JSON array
    avg_valence     REAL,
    avg_arousal     REAL,
    avg_loudness    REAL,
    timbral_centroid REAL,
    genre_clusters  TEXT,                  -- JSON: cluster weights
    taste_vector    TEXT,                  -- JSON: full high-dim vector
    -- Evolution metadata
    delta_from_prev TEXT                   -- JSON: what changed vs last profile
);

-- Generated tracks
CREATE TABLE generated_tracks (
    id              TEXT PRIMARY KEY,
    profile_id      TEXT NOT NULL REFERENCES taste_profiles(id),
    generated_at    DATETIME NOT NULL,
    prompt_used     TEXT,                  -- The MusicGen prompt
    duration_sec    REAL,
    file_path       TEXT,                  -- Local path to generated file
    model_version   TEXT,
    generation_params TEXT,               -- JSON: temperature, top_k, etc.
    user_rating     INTEGER,              -- 1-5, NULL if unrated
    user_feedback   TEXT,                 -- Optional text note
    is_favorite     BOOLEAN DEFAULT FALSE
);

-- Listening history (for taste evolution)
CREATE TABLE listening_events (
    id              TEXT PRIMARY KEY,
    track_id        TEXT REFERENCES tracks(id),
    generated_id    TEXT REFERENCES generated_tracks(id),
    listened_at     DATETIME NOT NULL,
    duration_heard  REAL,                  -- seconds actually listened
    completed       BOOLEAN,
    skipped_at      REAL                   -- seconds in when skipped
);
🧠 The Taste Vector System
Python

# taste_vector.py

@dataclass
class TasteVector:
    """
    A high-dimensional fingerprint of a user's musical taste.
    All values normalized to 0.0 - 1.0 range.
    """
    # === TEMPORAL ===
    preferred_bpm: float          # Weighted average BPM
    bpm_variance: float           # How varied their tempo preference is
    beat_regularity: float        # Preference for rigid vs. loose grooves
    rhythmic_complexity: float    # Simple 4/4 vs polyrhythmic

    # === HARMONIC ===
    key_brightness: float         # Dark (minor/low) vs bright (major/high)
    harmonic_complexity: float    # Simple chords vs complex harmony
    chroma_signature: List[float] # 12-dim: which pitch classes they love

    # === TIMBRAL ===
    brightness: float             # Dark vs bright timbres
    warmth: float                 # Cold (electronic) vs warm (acoustic)
    texture_roughness: float      # Smooth vs gritty/distorted
    density: float                # Sparse vs dense arrangements
    mfcc_signature: List[float]   # 20-dim timbral fingerprint

    # === ENERGY ===
    energy_level: float           # Quiet/calm vs loud/intense
    dynamic_range_pref: float     # Compressed vs dynamic
    loudness_preference: float    # LUFS preference

    # === MOOD ===
    valence: float                # Sad vs happy
    arousal: float                # Calm vs energetic

    # === META ===
    genre_cluster_weights: Dict[str, float]  # Soft genre assignments
    library_size: int
    profile_confidence: float     # How confident we are in this profile
    created_at: datetime

    def to_musicgen_prompt(self) -> str:
        """Convert taste vector to a natural language MusicGen prompt"""
        ...

    def distance_to(self, other: 'TasteVector') -> float:
        """Cosine distance between two taste vectors"""
        ...

    def evolve(self, new_features: List[FeatureVector],
               weight: float = 0.3) -> 'TasteVector':
        """Update taste vector with new listening data"""
        ...
🎼 Prompt Builder Logic
Python

# prompt_builder.py

class PromptBuilder:
    """
    Converts a TasteVector into a MusicGen conditioning prompt.
    MusicGen responds to natural language descriptions.
    """

    def build(self, vector: TasteVector, user_hint: str = "") -> str:
        parts = []

        # BPM
        if vector.preferred_bpm < 70:
            parts.append("slow, contemplative")
        elif vector.preferred_bpm < 100:
            parts.append("mid-tempo, relaxed groove")
        elif vector.preferred_bpm < 130:
            parts.append("upbeat, driving rhythm")
        else:
            parts.append("fast, energetic, high-BPM")

        # Mood
        if vector.valence > 0.7 and vector.arousal > 0.7:
            parts.append("euphoric, uplifting, joyful")
        elif vector.valence < 0.3 and vector.arousal < 0.3:
            parts.append("melancholic, quiet, introspective")
        elif vector.valence < 0.3 and vector.arousal > 0.7:
            parts.append("intense, dark, aggressive")
        elif vector.valence > 0.7 and vector.arousal < 0.3:
            parts.append("peaceful, warm, comforting")

        # Timbre
        if vector.warmth > 0.7:
            parts.append("warm acoustic instruments, organic feel")
        elif vector.warmth < 0.3:
            parts.append("electronic, synthesized, cold digital textures")

        if vector.texture_roughness > 0.6:
            parts.append("distorted, gritty, rough edges")

        # Genre clusters
        top_genres = sorted(
            vector.genre_cluster_weights.items(),
            key=lambda x: x[1], reverse=True
        )[:2]
        for genre, weight in top_genres:
            if weight > 0.2:
                parts.append(genre)

        # User override
        if user_hint:
            parts.append(user_hint)

        return ", ".join(parts)

    # Example output:
    # "mid-tempo, relaxed groove, melancholic, quiet, introspective,
    #  warm acoustic instruments, organic feel, indie folk, ambient"
🎛️ Generation Engine
Python

# generator/engine.py

class GenerationEngine:
    def __init__(self, model_path: str, device: str = "auto"):
        self.model = self._load_musicgen(model_path, device)
        self.prompt_builder = PromptBuilder()

    def _load_musicgen(self, path, device):
        from audiocraft.models import MusicGen
        model = MusicGen.get_pretrained(path)
        model.set_generation_params(
            use_sampling=True,
            top_k=250,
            duration=30  # seconds, configurable
        )
        return model

    def generate(
        self,
        profile: TasteProfile,
        duration_sec: int = 30,
        user_hint: str = "",
        seed_audio_path: str = None,   # Optional: continue from existing audio
        temperature: float = 1.0,
        variations: int = 1
    ) -> List[GeneratedTrack]:

        prompt = self.prompt_builder.build(
            profile.taste_vector,
            user_hint
        )

        # Optional: melody conditioning
        if seed_audio_path:
            melody, sr = torchaudio.load(seed_audio_path)
            wav = self.model.generate_with_chroma(
                descriptions=[prompt] * variations,
                melody_wavs=melody.unsqueeze(0).expand(variations, -1, -1),
                melody_sample_rate=sr,
                progress=True
            )
        else:
            wav = self.model.generate(
                descriptions=[prompt] * variations,
                progress=True
            )

        results = []
        for i, audio in enumerate(wav):
            track = self._save_and_register(audio, prompt, profile, i)
            results.append(track)

        return results

    def generate_playlist(
        self,
        profile: TasteProfile,
        track_count: int = 8,
        total_duration_min: int = 30
    ) -> List[GeneratedTrack]:
        """Generate a full playlist of varied tracks in one session"""
        ...
🖥️ Terminal UI (Textual)
text

┌──────────────────── SoundDNA v0.1.0 ──────────────────────┐
│  Library: 2,847 tracks  │  Profile confidence: 94%         │
│  Last generated: 12 min ago                                │
├─────────────────────────────────────────────────────────── │
│                                                            │
│  YOUR TASTE DNA                                            │
│  ──────────────                                            │
│  Tempo       ████████░░  ~112 BPM (varied)                 │
│  Energy      ██████░░░░  Medium-high                       │
│  Mood        ████████░░  Positive, energetic               │
│  Brightness  ████░░░░░░  Warm / dark                       │
│  Texture     ██░░░░░░░░  Smooth                            │
│  Complexity  ██████████  Highly complex harmonics          │
│                                                            │
│  Top vibes: indie electronic · jazz-adjacent · atmospheric │
│                                                            │
├────────────────────────────────────────────────────────────│
│  GENERATE NEW MUSIC                                        │
│  ─────────────────                                         │
│  Duration:  [30s] [60s] [90s] [Custom: _____]              │
│  Hint:      [something for late night coding_________]     │
│  Variations: [1] [3] [5]                                   │
│                                                            │
│  [▶ Generate]  [▶ Generate Playlist]  [⚙ Advanced]        │
│                                                            │
├────────────────────────────────────────────────────────────│
│  RECENT GENERATIONS                           [📁 Library] │
│  ─────────────────                                         │
│  ♪ sounddna_20250112_221034.wav   ★★★★☆  [▶] [💾] [🗑]   │
│  ♪ sounddna_20250112_195521.wav   ★★★☆☆  [▶] [💾] [🗑]   │
│  ♪ sounddna_20250112_181103.wav   Unrated  [▶] [💾] [🗑]  │
└────────────────────────────────────────────────────────────┘
⚙️ Configuration File
toml

# ~/.config/sounddna/config.toml

[library]
paths = [
    "~/Music",
    "/media/external/Music"
]
scan_on_startup = true
watch_for_changes = true
supported_formats = ["mp3", "flac", "wav", "m4a", "ogg", "aiff"]

[model]
name = "musicgen-small"        # or "musicgen-medium" (needs more VRAM)
path = "~/.sounddna/models/"
device = "auto"                # "auto" | "cuda" | "mps" | "cpu"
quantize = true                # 8-bit quantization for lower VRAM

[generation]
default_duration = 30          # seconds
default_variations = 1
output_dir = "~/Music/SoundDNA Generated"
output_format = "wav"          # "wav" | "flac" | "mp3"
normalize_output = true

[profile]
rebuild_interval_days = 7      # Re-analyze library every N days
min_tracks_for_profile = 20    # Minimum tracks needed
evolution_weight = 0.3         # How much new listening shifts the profile

[ui]
mode = "tui"                   # "tui" | "web"
web_port = 8899
theme = "dark"
📦 Tech Stack Summary
Layer	Technology
Language	Python 3.11+
Audio Analysis	librosa, essentia
Feature Caching	joblib
Dimensionality Reduction	UMAP-learn
Clustering	scikit-learn (K-means)
Music Generation	Facebook MusicGen (audiocraft)
Post-processing	pyloudnorm, pydub, soundfile
Metadata	mutagen
Database	SQLite (sqlite3 / aiosqlite)
Terminal UI	Textual
Web UI (opt.)	FastAPI + Svelte
GPU Inference	PyTorch + CUDA / MPS (Apple Silicon)
File Watching	watchdog
🚀 Build Phases
text

Phase 1 — Library Scanner + Feature Extractor (3 weeks)
├── Directory scanner
├── librosa feature pipeline
├── SQLite schema
└── Feature caching

Phase 2 — Taste Profile Engine (2 weeks)
├── UMAP reduction
├── K-means clustering
├── TasteVector builder
└── Evolution tracking

Phase 3 — Generation Engine (3 weeks)
├── MusicGen integration
├── Prompt builder
├── Generation session manager
└── Post-processing pipeline

Phase 4 — Terminal UI (2 weeks)
├── Textual TUI
├── Taste radar widget
├── Built-in player
└── Rating system

Phase 5 — Polish + Web UI (2 weeks)
├── FastAPI server
├── Svelte dashboard
├── Playlist generator
└── Export + sharing tools
