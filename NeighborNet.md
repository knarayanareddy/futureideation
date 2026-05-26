🌐 NeighborNet
Overview
A dead-simple toolkit for creating a local mesh network with a built-in local-only web — local Wikipedia, bulletin board, file sharing, chat, and classifieds — that works with zero internet access.

Architecture Overview
text

┌─────────────────────────────────────────────────────────────────┐
│                         NeighborNet                              │
├──────────────┬──────────────────┬────────────────────────────────┤
│  Mesh Layer  │  Local Web       │  Apps Layer                     │
│  (Yggdrasil  │  (Caddy reverse  │  (Wiki, Board, Files,           │
│   overlay)   │   proxy + DNS)   │   Chat, Market)                 │
└──────────────┴──────────────────┴────────────────────────────────┘
Full Component Breakdown
1. Project Structure
text

neighbornet/
├── setup/
│   ├── install.sh          # One-command installer
│   ├── wizard.py           # Interactive setup wizard
│   └── templates/
│       ├── yggdrasil.conf.j2
│       ├── caddy.j2
│       └── services.j2
├── core/
│   ├── mesh/
│   │   ├── node.py         # Node management
│   │   ├── discovery.py    # Peer discovery (mDNS)
│   │   └── monitor.py      # Network health monitoring
│   ├── dns/
│   │   └── resolver.py     # Local .neighbor DNS
│   └── dashboard/
│       ├── app.py          # Flask dashboard
│       └── templates/
├── apps/
│   ├── wiki/               # Local wiki (WikiJS or custom)
│   ├── board/              # Bulletin board (Flask)
│   ├── files/              # File sharing (Flask + local storage)
│   ├── chat/               # Real-time chat (WebSocket)
│   └── market/             # Local classifieds (Flask)
├── docker-compose.yml      # All apps as containers
├── requirements.txt
└── README.md
2. One-Command Installer
Bash

#!/bin/bash
# setup/install.sh
# Usage: curl -sSL https://neighbornet.local/install | bash

set -e

echo "🌐 NeighborNet Installer"
echo "========================"

# Detect OS
OS=$(uname -s)
ARCH=$(uname -m)

# Install dependencies
install_deps() {
    if command -v apt-get &>/dev/null; then
        sudo apt-get update -q
        sudo apt-get install -y -q \
            docker.io docker-compose \
            python3 python3-pip \
            avahi-daemon avahi-utils \
            net-tools iproute2 curl
    elif command -v brew &>/dev/null; then
        brew install docker docker-compose python3 avahi
    fi
}

# Install Yggdrasil
install_yggdrasil() {
    echo "📡 Installing Yggdrasil mesh network..."
    YGGDRASIL_VER="0.5.6"

    if [[ "$ARCH" == "x86_64" ]]; then
        wget -q "https://github.com/yggdrasil-network/yggdrasil-go/releases/download/v${YGGDRASIL_VER}/yggdrasil-${YGGDRASIL_VER}-x86_64.deb"
        sudo dpkg -i "yggdrasil-${YGGDRASIL_VER}-x86_64.deb"
    elif [[ "$ARCH" == "aarch64" ]]; then
        wget -q "https://github.com/yggdrasil-network/yggdrasil-go/releases/download/v${YGGDRASIL_VER}/yggdrasil-${YGGDRASIL_VER}-arm64.deb"
        sudo dpkg -i "yggdrasil-${YGGDRASIL_VER}-arm64.deb"
    fi
}

# Run interactive setup wizard
run_wizard() {
    python3 setup/wizard.py
}

install_deps
install_yggdrasil
run_wizard

echo ""
echo "✅ NeighborNet is running!"
echo "   Open: http://home.neighbor in your browser"
echo "   Your node address: $(yggdrasilctl getSelf | grep IPv6 | awk '{print $2}')"
3. Interactive Setup Wizard
Python

# setup/wizard.py
import subprocess
import json
import os
import socket
from pathlib import Path

def wizard():
    print("\n🌐 NeighborNet Setup Wizard")
    print("=" * 40)

    # Get node name
    hostname = socket.gethostname()
    node_name = input(f"Node name [{hostname}]: ").strip() or hostname

    # Get neighborhood name
    neighborhood = input("Neighborhood name [MyNeighborhood]: ").strip() or "MyNeighborhood"

    # WiFi or Ethernet?
    print("\nNetwork interface:")
    interfaces = get_network_interfaces()
    for i, iface in enumerate(interfaces):
        print(f"  {i+1}. {iface}")
    choice = int(input("Select interface [1]: ").strip() or "1") - 1
    interface = interfaces[choice]

    # Which apps to enable?
    print("\nSelect apps to enable (press Enter for all):")
    apps = {
        "wiki": input("  📚 Local Wiki? [Y/n]: ").strip().lower() != 'n',
        "board": input("  📋 Bulletin Board? [Y/n]: ").strip().lower() != 'n',
        "files": input("  📁 File Sharing? [Y/n]: ").strip().lower() != 'n',
        "chat": input("  💬 Chat? [Y/n]: ").strip().lower() != 'n',
        "market": input("  🛒 Local Market? [Y/n]: ").strip().lower() != 'n',
    }

    # Generate Yggdrasil config
    generate_yggdrasil_config(interface)

    # Generate docker-compose
    generate_docker_compose(neighborhood, node_name, apps)

    # Generate Caddy config
    generate_caddy_config(neighborhood, apps)

    # Generate DNS config
    generate_dns_config(neighborhood)

    # Start everything
    print("\n🚀 Starting NeighborNet...")
    subprocess.run(["sudo", "systemctl", "start", "yggdrasil"])
    subprocess.run(["docker-compose", "up", "-d"])

    print(f"\n✅ Done! Visit http://home.{neighborhood.lower()}.neighbor")

def get_network_interfaces():
    result = subprocess.run(["ip", "-o", "link", "show"],
                           capture_output=True, text=True)
    interfaces = []
    for line in result.stdout.splitlines():
        parts = line.split(':')
        if len(parts) >= 2:
            iface = parts[1].strip()
            if iface not in ['lo', 'docker0']:
                interfaces.append(iface)
    return interfaces

def generate_yggdrasil_config(interface: str):
    # Generate keys
    result = subprocess.run(["yggdrasil", "-genconf"],
                           capture_output=True, text=True)
    config = json.loads(result.stdout)

    # Set multicast interface for local peer discovery
    config["MulticastInterfaces"] = [{
        "Regex": interface,
        "Beacon": True,
        "Listen": True,
        "Port": 0,
        "Priority": 0
    }]

    config_path = Path("/etc/yggdrasil.conf")
    config_path.write_text(json.dumps(config, indent=2))
    print(f"✅ Yggdrasil config written to {config_path}")

def generate_docker_compose(neighborhood: str, node_name: str, apps: dict):
    compose = {
        "version": "3.8",
        "services": {},
        "networks": {
            "neighbornet": {"driver": "bridge"}
        }
    }

    # Always include dashboard and caddy
    compose["services"]["dashboard"] = {
        "image": "neighbornet/dashboard:latest",
        "build": "./core/dashboard",
        "environment": {
            "NEIGHBORHOOD": neighborhood,
            "NODE_NAME": node_name,
        },
        "networks": ["neighbornet"]
    }

    compose["services"]["caddy"] = {
        "image": "caddy:2",
        "ports": ["80:80", "443:443"],
        "volumes": [
            "./Caddyfile:/etc/caddy/Caddyfile",
            "caddy_data:/data"
        ],
        "networks": ["neighbornet"]
    }

    # Add enabled apps
    if apps.get("wiki"):
        compose["services"]["wiki"] = {
            "image": "requarks/wiki:2",
            "environment": {
                "DB_TYPE": "sqlite",
                "DB_FILEPATH": "/wiki/db.sqlite"
            },
            "volumes": ["wiki_data:/wiki"],
            "networks": ["neighbornet"]
        }

    if apps.get("board"):
        compose["services"]["board"] = {
            "build": "./apps/board",
            "volumes": ["board_data:/data"],
            "networks": ["neighbornet"]
        }

    if apps.get("files"):
        compose["services"]["files"] = {
            "build": "./apps/files",
            "volumes": ["files_data:/uploads"],
            "networks": ["neighbornet"]
        }

    if apps.get("chat"):
        compose["services"]["chat"] = {
            "build": "./apps/chat",
            "networks": ["neighbornet"]
        }

    if apps.get("market"):
        compose["services"]["market"] = {
            "build": "./apps/market",
            "volumes": ["market_data:/data"],
            "networks": ["neighbornet"]
        }

    Path("docker-compose.yml").write_text(
        "# Generated by NeighborNet Wizard\n" +
        json.dumps(compose, indent=2)
    )

if __name__ == "__main__":
    wizard()
4. Local DNS Resolution (.neighbor TLD)
Python

# core/dns/resolver.py
# A simple DNS server that resolves *.neighbor to local Yggdrasil IPs

import socket
import struct
import threading
from dnslib import DNSRecord, DNSHeader, RR, A, QTYPE
import subprocess

NEIGHBOR_TLD = ".neighbor"
LOCAL_IP = "127.0.0.1"
YGGDRASIL_IP = None  # Set on startup

def get_yggdrasil_ip() -> str:
    result = subprocess.run(
        ["yggdrasilctl", "getSelf"],
        capture_output=True, text=True
    )
    for line in result.stdout.splitlines():
        if "IPv6" in line:
            return line.split()[-1]
    return "::1"

class NeighborDNS:
    def __init__(self, port: int = 53):
        self.port = port
        self.records = {
            f"home.neighbor.": LOCAL_IP,
            f"wiki.neighbor.": LOCAL_IP,
            f"board.neighbor.": LOCAL_IP,
            f"files.neighbor.": LOCAL_IP,
            f"chat.neighbor.": LOCAL_IP,
            f"market.neighbor.": LOCAL_IP,
        }

    def start(self):
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        sock.bind(("0.0.0.0", self.port))
        print(f"🌐 NeighborNet DNS listening on :{self.port}")

        while True:
            data, addr = sock.recvfrom(1024)
            threading.Thread(
                target=self.handle_query,
                args=(sock, data, addr)
            ).start()

    def handle_query(self, sock, data: bytes, addr):
        request = DNSRecord.parse(data)
        reply = request.reply()

        qname = str(request.q.qname)

        if qname in self.records:
            reply.add_answer(RR(
                qname,
                QTYPE.A,
                rdata=A(self.records[qname]),
                ttl=60
            ))
        elif qname.endswith(NEIGHBOR_TLD + "."):
            # Default: resolve to self
            reply.add_answer(RR(
                qname,
                QTYPE.A,
                rdata=A(LOCAL_IP),
                ttl=60
            ))
        # else: forward to real DNS (for internet access if available)

        sock.sendto(reply.pack(), addr)

    def register_peer(self, hostname: str, ip: str):
        self.records[f"{hostname}.neighbor."] = ip
        print(f"📡 Registered peer: {hostname}.neighbor → {ip}")

if __name__ == "__main__":
    dns = NeighborDNS()
    dns.start()
5. Caddy Configuration (Reverse Proxy + Local TLS)
text

# Caddyfile
{
    # Use local CA for .neighbor TLS (self-signed, auto-trusted)
    local_certs
    log {
        output stdout
    }
}

home.neighbor {
    reverse_proxy dashboard:5000
    tls internal
}

wiki.neighbor {
    reverse_proxy wiki:3000
    tls internal
}

board.neighbor {
    reverse_proxy board:8001
    tls internal
}

files.neighbor {
    reverse_proxy files:8002
    tls internal
    # Increase upload limit for file sharing
    request_body {
        max_size 10GB
    }
}

chat.neighbor {
    reverse_proxy chat:8003
    tls internal
    # WebSocket support
    header {
        Connection *Upgrade*
        Upgrade websocket
    }
}

market.neighbor {
    reverse_proxy market:8004
    tls internal
}
6. Chat App (WebSocket-based)
Python

# apps/chat/app.py
from flask import Flask, render_template
from flask_socketio import SocketIO, emit, join_room, leave_room
from datetime import datetime
import sqlite3

app = Flask(__name__)
app.config['SECRET_KEY'] = 'neighbornet-secret'
socketio = SocketIO(app, cors_allowed_origins="*")

def init_db():
    conn = sqlite3.connect('chat.db')
    conn.execute('''
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            room TEXT NOT NULL,
            username TEXT NOT NULL,
            message TEXT NOT NULL,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    conn.close()

@app.route('/')
def index():
    return render_template('chat.html')

@app.route('/room/<room_name>')
def room(room_name):
    # Load last 50 messages
    conn = sqlite3.connect('chat.db')
    messages = conn.execute(
        'SELECT username, message, timestamp FROM messages '
        'WHERE room = ? ORDER BY timestamp DESC LIMIT 50',
        (room_name,)
    ).fetchall()
    conn.close()
    return render_template('room.html', room=room_name,
                          messages=reversed(messages))

@socketio.on('join')
def on_join(data):
    username = data['username']
    room = data['room']
    join_room(room)
    emit('status', {
        'msg': f'{username} has joined the room.',
        'timestamp': datetime.now().isoformat()
    }, room=room)

@socketio.on('message')
def on_message(data):
    username = data['username']
    room = data['room']
    message = data['message']

    # Save to DB
    conn = sqlite3.connect('chat.db')
    conn.execute(
        'INSERT INTO messages (room, username, message) VALUES (?, ?, ?)',
        (room, username, message)
    )
    conn.commit()
    conn.close()

    # Broadcast to room
    emit('message', {
        'username': username,
        'message': message,
        'timestamp': datetime.now().isoformat()
    }, room=room)

@socketio.on('leave')
def on_leave(data):
    username = data['username']
    room = data['room']
    leave_room(room)
    emit('status', {'msg': f'{username} has left.'}, room=room)

if __name__ == '__main__':
    init_db()
    socketio.run(app, host='0.0.0.0', port=8003)
7. Bulletin Board App
Python

# apps/board/app.py
from flask import Flask, render_template, request, redirect, url_for
import sqlite3
from datetime import datetime

app = Flask(__name__)

def init_db():
    conn = sqlite3.connect('board.db')
    conn.executescript('''
        CREATE TABLE IF NOT EXISTS posts (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            body TEXT NOT NULL,
            author TEXT NOT NULL DEFAULT 'anonymous',
            category TEXT DEFAULT 'general',
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            pinned BOOLEAN DEFAULT 0
        );

        CREATE TABLE IF NOT EXISTS comments (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            post_id INTEGER REFERENCES posts(id),
            body TEXT NOT NULL,
            author TEXT DEFAULT 'anonymous',
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        );
    ''')
    conn.commit()
    conn.close()

CATEGORIES = ['general', 'events', 'lost-and-found', 'help', 'news', 'humor']

@app.route('/')
def index():
    category = request.args.get('category', 'all')
    conn = sqlite3.connect('board.db')
    if category == 'all':
        posts = conn.execute(
            'SELECT * FROM posts ORDER BY pinned DESC, created_at DESC'
        ).fetchall()
    else:
        posts = conn.execute(
            'SELECT * FROM posts WHERE category = ? '
            'ORDER BY pinned DESC, created_at DESC',
            (category,)
        ).fetchall()
    conn.close()
    return render_template('index.html', posts=posts,
                          categories=CATEGORIES, current=category)

@app.route('/post', methods=['GET', 'POST'])
def new_post():
    if request.method == 'POST':
        title = request.form['title']
        body = request.form['body']
        author = request.form.get('author', 'anonymous')
        category = request.form.get('category', 'general')

        conn = sqlite3.connect('board.db')
        conn.execute(
            'INSERT INTO posts (title, body, author, category) VALUES (?, ?, ?, ?)',
            (title, body, author, category)
        )
        conn.commit()
        conn.close()
        return redirect(url_for('index'))

    return render_template('new_post.html', categories=CATEGORIES)

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=8001)
8. File Sharing App
Python

# apps/files/app.py
from flask import Flask, render_template, request, send_file, redirect
import os
import sqlite3
import hashlib
from pathlib import Path
from werkzeug.utils import secure_filename

app = Flask(__name__)
UPLOAD_FOLDER = Path('/uploads')
UPLOAD_FOLDER.mkdir(exist_ok=True)

def init_db():
    conn = sqlite3.connect('files.db')
    conn.execute('''
        CREATE TABLE IF NOT EXISTS files (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            filename TEXT NOT NULL,
            original_name TEXT NOT NULL,
            uploader TEXT DEFAULT 'anonymous',
            description TEXT,
            size INTEGER,
            sha256 TEXT,
            downloads INTEGER DEFAULT 0,
            uploaded_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    conn.close()

@app.route('/')
def index():
    conn = sqlite3.connect('files.db')
    files = conn.execute(
        'SELECT * FROM files ORDER BY uploaded_at DESC'
    ).fetchall()
    conn.close()
    return render_template('index.html', files=files)

@app.route('/upload', methods=['POST'])
def upload():
    file = request.files['file']
    description = request.form.get('description', '')
    uploader = request.form.get('uploader', 'anonymous')

    if file:
        original_name = file.filename
        safe_name = secure_filename(original_name)
        # Add hash prefix to avoid collisions
        content = file.read()
        file_hash = hashlib.sha256(content).hexdigest()[:8]
        stored_name = f"{file_hash}_{safe_name}"

        file_path = UPLOAD_FOLDER / stored_name
        file_path.write_bytes(content)

        conn = sqlite3.connect('files.db')
        conn.execute(
            'INSERT INTO files (filename, original_name, uploader, '
            'description, size, sha256) VALUES (?, ?, ?, ?, ?, ?)',
            (stored_name, original_name, uploader,
             description, len(content), file_hash)
        )
        conn.commit()
        conn.close()

    return redirect('/')

@app.route('/download/<int:file_id>')
def download(file_id):
    conn = sqlite3.connect('files.db')
    file = conn.execute(
        'SELECT * FROM files WHERE id = ?', (file_id,)
    ).fetchone()
    conn.execute(
        'UPDATE files SET downloads = downloads + 1 WHERE id = ?',
        (file_id,)
    )
    conn.commit()
    conn.close()

    return send_file(
        UPLOAD_FOLDER / file[1],
        as_attachment=True,
        download_name=file[2]
    )

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=8002)
9. Peer Discovery via mDNS
Python

# core/mesh/discovery.py
import socket
import threading
from zeroconf import ServiceInfo, Zeroconf, ServiceBrowser
import ipaddress

class PeerDiscovery:
    SERVICE_TYPE = "_neighbornet._tcp.local."

    def __init__(self, node_name: str, local_ip: str, port: int = 80):
        self.node_name = node_name
        self.local_ip = local_ip
        self.port = port
        self.peers = {}
        self.zeroconf = Zeroconf()

    def announce(self):
        info = ServiceInfo(
            self.SERVICE_TYPE,
            f"{self.node_name}.{self.SERVICE_TYPE}",
            addresses=[socket.inet_aton(self.local_ip)],
            port=self.port,
            properties={
                b"node": self.node_name.encode(),
                b"version": b"1.0",
            }
        )
        self.zeroconf.register_service(info)
        print(f"📡 Announced as {self.node_name} on local network")

    def discover(self, on_peer_found=None):
        class Listener:
            def add_service(self, zc, type_, name):
                info = zc.get_service_info(type_, name)
                if info:
                    peer = {
                        "name": name.split('.')[0],
                        "ip": socket.inet_ntoa(info.addresses[0]),
                        "port": info.port
                    }
                    self.peers[peer["name"]] = peer
                    print(f"🔍 Found peer: {peer['name']} at {peer['ip']}")
                    if on_peer_found:
                        on_peer_found(peer)

            def remove_service(self, zc, type_, name):
                peer_name = name.split('.')[0]
                self.peers.pop(peer_name, None)
                print(f"👋 Peer left: {peer_name}")

            def update_service(self, zc, type_, name):
                pass

        listener = Listener()
        browser = ServiceBrowser(self.zeroconf, self.SERVICE_TYPE, listener)
        return browser

    def shutdown(self):
        self.zeroconf.unregister_all_services()
        self.zeroconf.close()
10. Network Dashboard
Python

# core/dashboard/app.py
from flask import Flask, render_template, jsonify
import subprocess
import psutil
import requests

app = Flask(__name__)

@app.route('/')
def index():
    return render_template('dashboard.html')

@app.route('/api/network-stats')
def network_stats():
    # Get Yggdrasil peers
    try:
        result = subprocess.run(
            ["yggdrasilctl", "getPeers"],
            capture_output=True, text=True, timeout=5
        )
        peers = result.stdout
    except:
        peers = "unavailable"

    # Get local node info
    try:
        result = subprocess.run(
            ["yggdrasilctl", "getSelf"],
            capture_output=True, text=True, timeout=5
        )
        self_info = result.stdout
    except:
        self_info = "unavailable"

    # System stats
    net_io = psutil.net_io_counters()

    return jsonify({
        "peers": peers,
        "self": self_info,
        "bytes_sent": net_io.bytes_sent,
        "bytes_recv": net_io.bytes_recv,
        "cpu_percent": psutil.cpu_percent(),
        "memory_percent": psutil.virtual_memory().percent,
        "disk_percent": psutil.disk_usage('/').percent,
    })

@app.route('/api/apps-status')
def apps_status():
    apps = [
        {"name": "Wiki", "url": "http://wiki:3000", "slug": "wiki"},
        {"name": "Board", "url": "http://board:8001", "slug": "board"},
        {"name": "Files", "url": "http://files:8002", "slug": "files"},
        {"name": "Chat", "url": "http://chat:8003", "slug": "chat"},
        {"name": "Market", "url": "http://market:8004", "slug": "market"},
    ]

    for app_info in apps:
        try:
            r = requests.get(app_info["url"], timeout=2)
            app_info["status"] = "online" if r.status_code < 500 else "error"
        except:
            app_info["status"] = "offline"

    return jsonify(apps)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
11. Development Roadmap
text

Phase 1 (Weeks 1-3): Mesh Foundation
├── Yggdrasil integration + config generation
├── mDNS peer discovery
├── One-command installer script
└── Basic setup wizard

Phase 2 (Weeks 4-6): Core Apps
├── Docker-compose orchestration
├── Caddy reverse proxy + .neighbor DNS
├── Bulletin board app
└── File sharing app

Phase 3 (Weeks 7-9): Community Apps
├── Real-time chat (WebSocket)
├── Local wiki (WikiJS integration)
├── Local classifieds/market
└── Network dashboard

Phase 4 (Weeks 10-12): Polish
├── Raspberry Pi image build
├── Offline maps integration (via Kiwix)
├── Automated peer certificate trust
└── Documentation + release
