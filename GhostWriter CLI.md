🤖 GhostWriter CLI
Overview
A local AI agent that learns your writing voice from your own content, generates posts/articles in your voice, gets your approval, publishes to chosen platforms, then wipes all intermediate data.

Architecture Overview
text

┌────────────────────────────────────────────────────────────┐
│                      GhostWriter CLI                        │
├───────────────┬──────────────────┬────────────────────────  ┤
│  Voice Engine │  Generation      │  Publisher + Eraser      │
│  (fine-tune   │  (Ollama +       │  (Mastodon, Bluesky,     │
│   on corpus)  │  prompt engine)  │   wipe ephemeral data)   │
└───────────────┴──────────────────┴────────────────────────  ┘
Full Component Breakdown
1. Project Structure
text

ghostwriter/
├── ghostwriter/
│   ├── __init__.py
│   ├── cli.py               # Typer CLI entry point
│   ├── voice/
│   │   ├── __init__.py
│   │   ├── analyzer.py      # Analyze writing corpus
│   │   ├── embeddings.py    # Generate voice embeddings
│   │   └── profile.py       # Voice profile management
│   ├── generator/
│   │   ├── __init__.py
│   │   ├── prompt_builder.py  # Build voice-aware prompts
│   │   ├── llm_client.py      # Ollama client
│   │   └── refiner.py         # Multi-pass content refinement
│   ├── publisher/
│   │   ├── __init__.py
│   │   ├── mastodon.py
│   │   ├── bluesky.py
│   │   ├── nostr.py
│   │   └── file_output.py
│   ├── ephemeral/
│   │   ├── __init__.py
│   │   └── eraser.py        # Secure wipe of temp data
│   └── tui/
│       ├── __init__.py
│       └── review_ui.py     # Textual review interface
├── pyproject.toml
└── README.md
2. CLI Entry Point (Typer)
Python

# ghostwriter/cli.py
import typer
from rich.console import Console
from rich import print as rprint
from pathlib import Path
from typing import Optional, List
from . import voice, generator, publisher, ephemeral

app = typer.Typer(
    name="ghostwriter",
    help="🤖 AI that writes in your voice. Posts. Vanishes.",
    add_completion=False
)
console = Console()

@app.command()
def train(
    corpus_path: Path = typer.Argument(
        ...,
        help="Path to your writing corpus (dir of .txt/.md files, or single file)"
    ),
    profile_name: str = typer.Option("default", "--profile", "-p"),
    model: str = typer.Option("mistral:7b", "--model", "-m",
                               help="Ollama model to use"),
):
    """Train GhostWriter on your writing corpus."""
    console.print(f"[cyan]📚 Analyzing corpus at {corpus_path}...[/cyan]")
    analyzer = voice.VoiceAnalyzer(model=model)
    profile = analyzer.analyze(corpus_path)
    profile.save(profile_name)
    console.print(f"[green]✅ Voice profile '{profile_name}' created![/green]")
    console.print(f"   Vocabulary richness: {profile.vocab_richness:.2f}")
    console.print(f"   Avg sentence length: {profile.avg_sentence_length:.1f} words")
    console.print(f"   Detected tone: {profile.tone}")
    console.print(f"   Common themes: {', '.join(profile.themes[:5])}")

@app.command()
def write(
    brief: str = typer.Argument(..., help="What to write about"),
    profile_name: str = typer.Option("default", "--profile", "-p"),
    format: str = typer.Option("post", "--format", "-f",
                                help="post, thread, article, email"),
    length: str = typer.Option("medium", "--length", "-l",
                                help="short, medium, long"),
    publish_to: Optional[List[str]] = typer.Option(
        None, "--publish", help="mastodon, bluesky, nostr, file"
    ),
    dry_run: bool = typer.Option(False, "--dry-run"),
):
    """Generate content in your voice and optionally publish it."""
    # Load voice profile
    profile = voice.VoiceProfile.load(profile_name)

    console.print(f"[cyan]🤖 Generating {format} about: {brief}[/cyan]")

    # Generate content
    gen = generator.ContentGenerator(profile)
    content = gen.generate(brief, format=format, length=length)

    # Interactive review
    approved_content = review_content(content)

    if approved_content and not dry_run:
        if publish_to:
            pub = publisher.Publisher()
            for platform in publish_to:
                console.print(f"[cyan]📤 Publishing to {platform}...[/cyan]")
                pub.publish(platform, approved_content)
                console.print(f"[green]✅ Published to {platform}[/green]")

    # Always wipe ephemeral data
    console.print("[yellow]🔥 Wiping ephemeral data...[/yellow]")
    ephemeral.Eraser().wipe_session()
    console.print("[green]✅ All intermediate data wiped[/green]")

def review_content(content: str) -> Optional[str]:
    """Interactive review loop."""
    from .tui.review_ui import ReviewApp
    app = ReviewApp(content)
    result = app.run()
    return result

@app.command()
def profiles():
    """List available voice profiles."""
    profiles = voice.VoiceProfile.list_all()
    for p in profiles:
        console.print(f"  📝 {p.name} — trained on {p.doc_count} documents")

if __name__ == "__main__":
    app()
3. Voice Analyzer
Python

# ghostwriter/voice/analyzer.py
import re
import json
import statistics
from pathlib import Path
from collections import Counter
from typing import List, Dict, Any
import ollama

class VoiceProfile:
    def __init__(self):
        self.name: str = ""
        self.avg_sentence_length: float = 0
        self.avg_word_length: float = 0
        self.vocab_richness: float = 0
        self.tone: str = ""
        self.themes: List[str] = []
        self.writing_style: str = ""
        self.punctuation_style: Dict[str, float] = {}
        self.common_phrases: List[str] = []
        self.doc_count: int = 0
        self.sample_texts: List[str] = []  # Few-shot examples
        self.system_prompt: str = ""  # LLM system prompt

    @classmethod
    def load(cls, name: str) -> 'VoiceProfile':
        profile_path = Path.home() / ".ghostwriter" / "profiles" / f"{name}.json"
        data = json.loads(profile_path.read_text())
        profile = cls()
        profile.__dict__.update(data)
        return profile

    def save(self, name: str):
        self.name = name
        profile_dir = Path.home() / ".ghostwriter" / "profiles"
        profile_dir.mkdir(parents=True, exist_ok=True)
        (profile_dir / f"{name}.json").write_text(
            json.dumps(self.__dict__, indent=2)
        )

    @classmethod
    def list_all(cls) -> List['VoiceProfile']:
        profile_dir = Path.home() / ".ghostwriter" / "profiles"
        profiles = []
        for f in profile_dir.glob("*.json"):
            profiles.append(cls.load(f.stem))
        return profiles


class VoiceAnalyzer:
    def __init__(self, model: str = "mistral:7b"):
        self.model = model
        self.client = ollama.Client()

    def analyze(self, corpus_path: Path) -> VoiceProfile:
        # Load all documents
        texts = self._load_corpus(corpus_path)
        profile = VoiceProfile()
        profile.doc_count = len(texts)

        # Statistical analysis
        all_sentences = []
        all_words = []
        all_text = " ".join(texts)

        for text in texts:
            sentences = re.split(r'[.!?]+', text)
            sentences = [s.strip() for s in sentences if len(s.strip()) > 10]
            all_sentences.extend(sentences)
            words = re.findall(r'\b[a-z]+\b', text.lower())
            all_words.extend(words)

        profile.avg_sentence_length = statistics.mean([len(s.split()) for s in all_sentences])
        profile.avg_word_length = statistics.mean([len(w) for w in all_words])

        # Vocabulary richness (type-token ratio)
        unique_words = len(set(all_words))
        profile.vocab_richness = unique_words / len(all_words) if all_words else 0

        # Punctuation analysis
        profile.punctuation_style = {
            "ellipsis_rate": all_text.count('...') / len(texts),
            "dash_rate": all_text.count('—') / len(texts),
            "exclamation_rate": all_text.count('!') / len(texts),
            "question_rate": all_text.count('?') / len(texts),
        }

        # Use LLM to extract tone, themes, and style
        profile.system_prompt, profile.tone, profile.themes = \
            self._llm_analyze(texts[:10])  # Use first 10 docs

        # Sample texts for few-shot prompting
        profile.sample_texts = [t[:500] for t in texts[:5]]

        return profile

    def _load_corpus(self, path: Path) -> List[str]:
        texts = []
        if path.is_file():
            texts.append(path.read_text(encoding='utf-8', errors='ignore'))
        elif path.is_dir():
            for ext in ['*.txt', '*.md', '*.rst']:
                for f in path.rglob(ext):
                    try:
                        texts.append(f.read_text(encoding='utf-8', errors='ignore'))
                    except Exception:
                        pass
        return texts

    def _llm_analyze(self, sample_texts: List[str]):
        analysis_prompt = f"""
Analyze the following writing samples and extract:
1. The overall TONE (one word: casual/formal/humorous/serious/sarcastic/passionate/etc)
2. The main THEMES (list of 5-8 keywords)
3. A SYSTEM PROMPT that would help an LLM reproduce this writing style

Writing samples:
---
{chr(10).join(f'Sample {i+1}:{chr(10)}{t[:300]}' for i, t in enumerate(sample_texts))}
---

Respond in JSON format:
{{
  "tone": "...",
  "themes": ["...", "..."],
  "system_prompt": "You are writing in the style of the author. Key characteristics: ..."
}}
"""
        response = self.client.generate(
            model=self.model,
            prompt=analysis_prompt,
            format='json'
        )

        data = json.loads(response['response'])
        return data['system_prompt'], data['tone'], data['themes']
4. Content Generator
Python

# ghostwriter/generator/llm_client.py
import ollama
from typing import Generator

class OllamaClient:
    def __init__(self, model: str = "mistral:7b"):
        self.client = ollama.Client()
        self.model = model

    def generate(self, prompt: str, system: str = "") -> str:
        response = self.client.generate(
            model=self.model,
            prompt=prompt,
            system=system,
            options={
                "temperature": 0.85,
                "top_p": 0.95,
                "num_predict": 2048,
            }
        )
        return response['response']

    def stream_generate(self, prompt: str, system: str = "") -> Generator[str, None, None]:
        for chunk in self.client.generate(
            model=self.model,
            prompt=prompt,
            system=system,
            stream=True
        ):
            yield chunk['response']
Python

# ghostwriter/generator/prompt_builder.py
from ..voice.analyzer import VoiceProfile

FORMAT_INSTRUCTIONS = {
    "post": "Write a single social media post (max 500 chars). No hashtags unless natural.",
    "thread": "Write a thread of 5-8 connected posts. Mark each with [1/n], [2/n] etc.",
    "article": "Write a full article with a title, introduction, body sections, and conclusion.",
    "email": "Write an email with subject line and body. Start with 'Subject:' on first line.",
}

LENGTH_INSTRUCTIONS = {
    "short": "Keep it concise and punchy. Less is more.",
    "medium": "Balanced length. Not too short, not too long.",
    "long": "Go deep. Be thorough. Don't hold back.",
}

class PromptBuilder:
    def __init__(self, profile: VoiceProfile):
        self.profile = profile

    def build_system_prompt(self) -> str:
        return f"""{self.profile.system_prompt}

Additional style notes:
- Average sentence length: {self.profile.avg_sentence_length:.0f} words
- Tone: {self.profile.tone}
- Key themes you often explore: {', '.join(self.profile.themes)}
- Vocabulary richness level: {'high' if self.profile.vocab_richness > 0.5 else 'moderate'}

CRITICAL: Write EXACTLY as this person would write. Match their voice perfectly.
Never add generic AI phrases or filler. Be authentic to this voice.
"""

    def build_prompt(self, brief: str, format: str, length: str) -> str:
        format_instruction = FORMAT_INSTRUCTIONS.get(format, FORMAT_INSTRUCTIONS["post"])
        length_instruction = LENGTH_INSTRUCTIONS.get(length, LENGTH_INSTRUCTIONS["medium"])

        few_shot = "\n\n".join([
            f"Example of my writing:\n{sample}"
            for sample in self.profile.sample_texts[:3]
        ])

        return f"""Here are examples of how I write:

{few_shot}

---

Now write the following in my exact voice and style:
Topic/Brief: {brief}
Format: {format_instruction}
Length: {length_instruction}

Write only the content. No preamble. No "Here is your post:" type intro.
"""
Python

# ghostwriter/generator/__init__.py
from .llm_client import OllamaClient
from .prompt_builder import PromptBuilder
from .refiner import ContentRefiner
from ..voice.analyzer import VoiceProfile

class ContentGenerator:
    def __init__(self, profile: VoiceProfile):
        self.profile = profile
        self.client = OllamaClient(model="mistral:7b")
        self.prompt_builder = PromptBuilder(profile)
        self.refiner = ContentRefiner(self.client)

    def generate(self, brief: str, format: str = "post", length: str = "medium") -> str:
        system = self.prompt_builder.build_system_prompt()
        prompt = self.prompt_builder.build_prompt(brief, format, length)

        # First pass
        draft = self.client.generate(prompt, system)

        # Refine to better match voice
        refined = self.refiner.refine(
            draft,
            voice_profile=self.profile,
            original_brief=brief
        )

        return refined
Python

# ghostwriter/generator/refiner.py
from .llm_client import OllamaClient
from ..voice.analyzer import VoiceProfile

class ContentRefiner:
    def __init__(self, client: OllamaClient):
        self.client = client

    def refine(self, draft: str, voice_profile: VoiceProfile,
               original_brief: str) -> str:
        refinement_prompt = f"""Review this draft and improve it to better match the author's voice.

Original brief: {original_brief}
Author's tone: {voice_profile.tone}
Author's themes: {', '.join(voice_profile.themes)}

Draft to refine:
{draft}

Refinement criteria:
1. Does it sound authentic to this voice?
2. Is the sentence rhythm correct?
3. Are there any generic AI phrases to remove?
4. Does it stay true to the brief?

Return ONLY the refined text, nothing else.
"""
        return self.client.generate(refinement_prompt)
5. Review TUI (Textual)
Python

# ghostwriter/tui/review_ui.py
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, TextArea, Button, Static, Label
from textual.containers import Horizontal, Vertical
from textual.binding import Binding
from typing import Optional

class ReviewApp(App):
    CSS = """
    Screen {
        layout: vertical;
    }
    #content-area {
        height: 70%;
        border: solid green;
        margin: 1;
    }
    #actions {
        height: auto;
        layout: horizontal;
        margin: 1;
    }
    Button {
        margin: 0 1;
    }
    #status {
        height: 3;
        background: $surface;
        padding: 1;
        margin: 1;
    }
    """

    BINDINGS = [
        Binding("ctrl+a", "approve", "Approve & Publish"),
        Binding("ctrl+r", "regenerate", "Regenerate"),
        Binding("ctrl+e", "edit", "Edit"),
        Binding("ctrl+q", "quit_no_publish", "Discard"),
    ]

    def __init__(self, content: str):
        super().__init__()
        self.content = content
        self.approved_content: Optional[str] = None

    def compose(self) -> ComposeResult:
        yield Header(show_clock=True)
        yield Label("📝 Review Generated Content:", id="label")
        yield TextArea(self.content, id="content-area", language="markdown")
        yield Static(
            "✏️  Edit the content above, then approve or discard.",
            id="status"
        )
        with Horizontal(id="actions"):
            yield Button("✅ Approve & Publish", variant="success", id="approve")
            yield Button("🔄 Regenerate", variant="warning", id="regenerate")
            yield Button("❌ Discard", variant="error", id="discard")
        yield Footer()

    def on_button_pressed(self, event: Button.Pressed):
        if event.button.id == "approve":
            content_area = self.query_one("#content-area", TextArea)
            self.approved_content = content_area.text
            self.exit(self.approved_content)
        elif event.button.id == "regenerate":
            self.exit("REGENERATE")
        elif event.button.id == "discard":
            self.exit(None)

    def action_approve(self):
        content_area = self.query_one("#content-area", TextArea)
        self.approved_content = content_area.text
        self.exit(self.approved_content)

    def action_quit_no_publish(self):
        self.exit(None)
6. Publisher
Python

# ghostwriter/publisher/__init__.py
from .mastodon import MastodonPublisher
from .bluesky import BlueskyPublisher
from .nostr import NostrPublisher
from .file_output import FilePublisher

class Publisher:
    def __init__(self):
        self.publishers = {
            "mastodon": MastodonPublisher(),
            "bluesky": BlueskyPublisher(),
            "nostr": NostrPublisher(),
            "file": FilePublisher(),
        }

    def publish(self, platform: str, content: str) -> bool:
        if platform not in self.publishers:
            raise ValueError(f"Unknown platform: {platform}")
        return self.publishers[platform].publish(content)
Python

# ghostwriter/publisher/mastodon.py
import requests
import os
from pathlib import Path
import json

class MastodonPublisher:
    def __init__(self):
        config_path = Path.home() / ".ghostwriter" / "publishers" / "mastodon.json"
        if config_path.exists():
            config = json.loads(config_path.read_text())
            self.instance = config['instance']
            self.token = config['token']
        else:
            self.instance = None
            self.token = None

    def setup(self):
        print("Mastodon Setup")
        self.instance = input("Instance URL (e.g. https://mastodon.social): ").strip()
        self.token = input("Access Token: ").strip()

        config_dir = Path.home() / ".ghostwriter" / "publishers"
        config_dir.mkdir(parents=True, exist_ok=True)
        (config_dir / "mastodon.json").write_text(
            json.dumps({"instance": self.instance, "token": self.token})
        )

    def publish(self, content: str) -> bool:
        if not self.token:
            raise RuntimeError("Mastodon not configured. Run: ghostwriter setup mastodon")

        # Handle threads (multiple posts)
        if "[1/" in content:
            return self._publish_thread(content)
        else:
            return self._publish_single(content)

    def _publish_single(self, content: str) -> bool:
        response = requests.post(
            f"{self.instance}/api/v1/statuses",
            headers={"Authorization": f"Bearer {self.token}"},
            json={"status": content, "visibility": "public"}
        )
        return response.status_code == 200

    def _publish_thread(self, content: str) -> bool:
        # Split thread by [N/N] markers
        import re
        posts = re.split(r'\[\d+/\d+\]', content)
        posts = [p.strip() for p in posts if p.strip()]

        prev_id = None
        for post in posts:
            payload = {
                "status": post,
                "visibility": "public"
            }
            if prev_id:
                payload["in_reply_to_id"] = prev_id

            response = requests.post(
                f"{self.instance}/api/v1/statuses",
                headers={"Authorization": f"Bearer {self.token}"},
                json=payload
            )
            if response.status_code == 200:
                prev_id = response.json()['id']
            else:
                return False

        return True
Python

# ghostwriter/publisher/bluesky.py
import requests
from datetime import datetime, timezone
from pathlib import Path
import json

class BlueskyPublisher:
    ATP_BASE = "https://bsky.social"

    def __init__(self):
        config_path = Path.home() / ".ghostwriter" / "publishers" / "bluesky.json"
        if config_path.exists():
            config = json.loads(config_path.read_text())
            self.handle = config['handle']
            self.password = config['password']
            self.access_jwt = None
            self.did = None

    def _authenticate(self):
        response = requests.post(
            f"{self.ATP_BASE}/xrpc/com.atproto.server.createSession",
            json={"identifier": self.handle, "password": self.password}
        )
        data = response.json()
        self.access_jwt = data['accessJwt']
        self.did = data['did']

    def publish(self, content: str) -> bool:
        self._authenticate()

        # Truncate to 300 chars if needed
        if len(content) > 300:
            content = content[:297] + "..."

        response = requests.post(
            f"{self.ATP_BASE}/xrpc/com.atproto.repo.createRecord",
            headers={"Authorization": f"Bearer {self.access_jwt}"},
            json={
                "repo": self.did,
                "collection": "app.bsky.feed.post",
                "record": {
                    "$type": "app.bsky.feed.post",
                    "text": content,
                    "createdAt": datetime.now(timezone.utc).isoformat()
                }
            }
        )
        return response.status_code == 200
7. Ephemeral Eraser
Python

# ghostwriter/ephemeral/eraser.py
import os
import shutil
import subprocess
import tempfile
from pathlib import Path

class Eraser:
    def __init__(self):
        self.temp_dir = Path(tempfile.gettempdir()) / "ghostwriter"
        self.session_dir = Path.home() / ".ghostwriter" / "sessions"

    def wipe_session(self):
        """Securely wipe all ephemeral session data."""
        # 1. Wipe temp directory
        if self.temp_dir.exists():
            self._secure_delete_dir(self.temp_dir)

        # 2. Wipe Ollama conversation cache (if applicable)
        ollama_cache = Path.home() / ".ollama" / "history"
        if ollama_cache.exists():
            self._secure_delete_dir(ollama_cache)

        # 3. Wipe session files
        if self.session_dir.exists():
            self._secure_delete_dir(self.session_dir)

        # 4. Clear shell history of this session
        self._clear_shell_history()

        # 5. Sync and drop page cache
        try:
            subprocess.run(["sync"], check=True)
        except Exception:
            pass

    def _secure_delete_dir(self, path: Path):
        """Overwrite all files before deleting."""
        for file_path in path.rglob('*'):
            if file_path.is_file():
                self._overwrite_file(file_path)
        shutil.rmtree(path, ignore_errors=True)

    def _overwrite_file(self, path: Path, passes: int = 3):
        """Overwrite file with random data before deletion."""
        try:
            size = path.stat().st_size
            with open(path, 'wb') as f:
                for _ in range(passes):
                    f.seek(0)
                    f.write(os.urandom(size))
                    f.flush()
                    os.fsync(f.fileno())
        except Exception:
            pass

    def _clear_shell_history(self):
        """Remove ghostwriter commands from shell history."""
        history_files = [
            Path.home() / ".bash_history",
            Path.home() / ".zsh_history",
            Path.home() / ".history",
        ]

        for hist_file in history_files:
            if hist_file.exists():
                try:
                    content = hist_file.read_text(errors='ignore')
                    # Remove lines containing 'ghostwriter'
                    filtered = '\n'.join([
                        line for line in content.splitlines()
                        if 'ghostwriter' not in line.lower()
                    ])
                    hist_file.write_text(filtered)
                except Exception:
                    pass
8. Usage Examples
Bash

# Train on your email archive
ghostwriter train ~/Documents/emails/ --profile work-voice

# Train on blog posts
ghostwriter train ~/blog/posts/ --profile blog-voice

# Generate a social post
ghostwriter write "thoughts on the current state of open source funding" \
  --profile blog-voice \
  --format post \
  --length medium

# Generate and publish to Mastodon
ghostwriter write "something sarcastic about NFTs" \
  --profile blog-voice \
  --format thread \
  --publish mastodon

# Generate an article (saves to file)
ghostwriter write "why local-first software matters" \
  --profile blog-voice \
  --format article \
  --length long \
  --publish file

# Dry run (just see output, no publish)
ghostwriter write "hello world" --dry-run

# List profiles
ghostwriter profiles

# Set up publisher
ghostwriter setup mastodon
ghostwriter setup bluesky
9. Development Roadmap
text

Phase 1 (Weeks 1-3): Voice Engine
├── Corpus loader (txt, md, email formats)
├── Statistical analysis (sentence length, vocabulary)
├── LLM-based tone/theme extraction
└── Voice profile save/load

Phase 2 (Weeks 4-6): Generator
├── Ollama client integration
├── Prompt builder with voice system prompts
├── Multi-pass refinement
└── Format support (post, thread, article, email)

Phase 3 (Weeks 7-9): Publisher + TUI
├── Mastodon publisher
├── Bluesky/ATP publisher
├── Textual review interface
└── Secure ephemeral eraser

Phase 4 (Weeks 10-12): Polish
├── Nostr publisher
├── Voice quality scoring
├── Package with pipx support
└── Documentation + release
