# Setup Guide

> Detailed installation, prerequisites, and configuration reference for the Firm ecosystem.

⚠️ Contenu généré par IA — validation humaine requise avant utilisation.

---

## Prerequisites

### Required Software

| Software | Minimum Version | Purpose |
|----------|----------------|---------|
| **Python** | 3.11+ | Both MCP servers |
| **Git** | 2.30+ | Version control |
| **VS Code** | 1.96+ | Agent mode support |
| **GitHub Copilot** | Latest | Agent mode + MCP client |
| **Node.js** | 18+ | ClawHub CLI (`npx clawhub`) |

### Optional Software

| Software | Version | Purpose |
|----------|---------|---------|
| **Docker** | 24+ | Container deployment |
| **Docker Compose** | 2.0+ | Multi-container orchestration |
| **Tesseract OCR** | 5.0+ | Image text extraction (Memory-os-ai) |
| **FFmpeg** | 6.0+ | Audio transcription (Memory-os-ai) |
| **Antiword** | — | Legacy .doc support |

### System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 4 GB | 8 GB+ |
| Disk | 2 GB | 5 GB+ (includes FAISS models) |
| CPU | 2 cores | 4+ cores |
| GPU | Not required | CUDA for faiss-gpu (optional) |

---

## Installation

### Step 1: Clone Repositories

```bash
# Choose a workspace directory
mkdir ~/ai-firm && cd ~/ai-firm

# Clone the parent repo (includes mcp-openclaw-extensions as subdirectory)
git clone https://github.com/romainsantoli-web/setup-vs-agent-firm.git

# Clone Memory-os-ai alongside it
git clone https://github.com/romainsantoli-web/Memory-os-ai.git

# Resulting structure:
# ~/ai-firm/
# ├── setup-vs-agent-firm/
# │   ├── mcp-openclaw-extensions/
# │   └── ...
# └── Memory-os-ai/
```

### Step 2: Setup mcp-openclaw-extensions

```bash
cd setup-vs-agent-firm/mcp-openclaw-extensions

# Create virtual environment
python3.11 -m venv .venv
source .venv/bin/activate  # or .venv\Scripts\activate on Windows

# Install dependencies
pip install -r requirements.txt

# Copy and configure environment
cp .env.example .env
```

Edit `.env` with your settings:

```bash
# .env — mcp-openclaw-extensions configuration
MCP_EXT_HOST=127.0.0.1          # Listen address
MCP_EXT_PORT=8012                # Listen port
LOG_LEVEL=INFO                   # DEBUG, INFO, WARNING, ERROR

# OpenClaw Gateway connection (optional)
OPENCLAW_GATEWAY_URL=ws://127.0.0.1:18789
OPENCLAW_GATEWAY_HTTP=http://127.0.0.1:18789
OPENCLAW_TIMEOUT_SECONDS=15

# VS Bridge settings
VS_BRIDGE_MAX_CONTEXT_BYTES=32768

# Fleet management
FLEET_CONFIG_PATH=~/.openclaw/fleet.json
FLEET_MAX_INSTANCES=50

# Delivery exports
FIRM_EXPORT_OUTPUT_DIR=~/.openclaw/exports

# Authentication (optional — leave empty to disable)
MCP_AUTH_TOKEN=

# Tool timeout (seconds)
TOOL_TIMEOUT_S=120

# API tokens for delivery export tools (optional)
# GITHUB_TOKEN=ghp_...
# JIRA_API_TOKEN=...
# LINEAR_API_KEY=lin_api_...
```

Verify the server starts:

```bash
python -m src.main
# Expected output:
# INFO: MCP Extensions server v3.3.0 starting on 127.0.0.1:8012
# INFO: Registered 138 tools from 29 modules
# INFO: MCP protocol version: 2025-11-25
```

### Step 3: Setup Memory-os-ai

```bash
cd ~/ai-firm/Memory-os-ai

# Create virtual environment
python3.11 -m venv .venv
source .venv/bin/activate

# Install with development dependencies
pip install -e ".[dev]"

# Install system dependencies for document processing
# macOS:
brew install tesseract ffmpeg antiword

# Ubuntu/Debian:
# sudo apt install tesseract-ocr ffmpeg antiword
```

Configure environment:

```bash
# Environment variables (add to shell config or .env)
export MEMORY_WORKSPACE=~/ai-firm        # Root workspace
export MEMORY_CACHE_DIR=~/.memory-os-ai  # FAISS index + cache
export MEMORY_MODEL=all-MiniLM-L6-v2     # Embedding model
export TOKENIZERS_PARALLELISM=false       # Avoid parallelism warnings

# Optional authentication for SSE/HTTP modes
# export MEMORY_API_KEY=your-secret-key
```

Verify the server starts:

```bash
# stdio mode (default — for Claude Code / VS Code)
memory-os-ai
# Or SSE mode on port 8765
memory-os-ai --sse
# Or Streamable HTTP mode on port 8765
memory-os-ai --http
```

### Step 4: Configure VS Code

Create or update `.vscode/mcp.json` in your project:

```json
{
  "servers": {
    "memory-os-ai": {
      "type": "stdio",
      "command": "/path/to/Memory-os-ai/.venv/bin/memory-os-ai",
      "env": {
        "MEMORY_WORKSPACE": "${workspaceFolder}",
        "MEMORY_CACHE_DIR": "${env:HOME}/.memory-os-ai"
      }
    },
    "openclaw-extensions": {
      "type": "http",
      "url": "http://127.0.0.1:8012/mcp"
    }
  }
}
```

Or copy the pre-configured file:

```bash
cp setup-vs-agent-firm/mcp-config-unified.json .vscode/mcp.json
```

### Step 5: Verify Connection

1. Open VS Code
2. Open the Copilot chat panel
3. Switch to **Agent mode** (dropdown at top)
4. Type: `@memory-os-ai memory_status`
5. Type: Ask any agent to call an openclaw tool

---

## Docker Deployment

### Quick Start

```bash
cd setup-vs-agent-firm

# Build and start both containers
docker compose up -d --build

# Check status
docker compose ps
docker compose logs -f
```

### docker-compose.yml Reference

```yaml
services:
  memory-os-ai:
    build:
      context: ../Memory-os-ai
      dockerfile: Dockerfile
    ports:
      - "8765:8765"
    volumes:
      - memory-data:/data
      - ./workspace:/workspace
    environment:
      - MEMORY_WORKSPACE=/workspace
      - MEMORY_CACHE_DIR=/data/.cache
    healthcheck:
      test: ["CMD", "python", "-c",
        "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8765/health')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  openclaw-extensions:
    build:
      context: ./mcp-openclaw-extensions
      dockerfile: Dockerfile
    ports:
      - "8012:8012"
    env_file:
      - ./mcp-openclaw-extensions/.env
    volumes:
      - ./workspace:/workspace
    healthcheck:
      test: ["CMD", "python", "-c",
        "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8012/health')"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 15s

volumes:
  memory-data:
```

### Docker Health Checks

```bash
# Check individual services
curl http://127.0.0.1:8012/health   # {"status": "ok", "version": "3.3.0"}
curl http://127.0.0.1:8765/health   # {"status": "ok"}

# Docker health
docker compose ps
# NAME                  STATUS
# memory-os-ai          Up (healthy)
# openclaw-extensions    Up (healthy)
```

---

## Firm Generation

### Generate Your First Firm

```bash
cd setup-vs-agent-firm

# See all options
bash factory/generate-firm.sh --help

# Generate a legal startup firm
bash factory/generate-firm.sh \
  --name "LegalAI" \
  --sector legal \
  --size startup \
  --stack python \
  --output ~/projects/legal-ai

# Generate an enterprise SaaS firm
bash factory/generate-firm.sh \
  --name "SaaSCorp" \
  --sector saas \
  --size enterprise \
  --stack typescript \
  --output ~/projects/saas-corp
```

### Install Skills

After generating, install the recommended skills:

```bash
cd ~/projects/legal-ai
bash scripts/install-skills.sh
# This runs: npx clawhub@latest install firm-orchestration
#             npx clawhub@latest install firm-legal-pack
#             npx clawhub@latest install firm-security-audit
#             ... (sector-specific skills)
```

### Use the Firm in VS Code

1. Open the generated project in VS Code
2. The agents are auto-discovered from `.github/agents/`
3. The prompts are auto-discovered from `.github/prompts/`
4. Switch to Agent mode in Copilot Chat
5. Use `@firm-ceo` to start — it delegates to department heads

---

## Auto-Setup for AI Clients (Memory-os-ai)

Memory-os-ai includes an auto-setup command for 5 AI clients:

```bash
# Setup all clients at once
memory-os-ai setup all

# Or one at a time
memory-os-ai setup claude-code     # Creates .claude/mcp.json
memory-os-ai setup claude-desktop  # Updates Claude Desktop config
memory-os-ai setup vscode          # Creates .vscode/mcp.json
memory-os-ai setup codex           # Creates AGENTS.md
memory-os-ai setup chatgpt         # Prints manual bridge instructions

# Check status
memory-os-ai setup status
```

### Generated Configuration Files

**Claude Code** (`.claude/mcp.json`):
```json
{
  "mcpServers": {
    "memory-os-ai": {
      "command": "memory-os-ai",
      "args": [],
      "env": {
        "MEMORY_WORKSPACE": "/path/to/workspace"
      }
    }
  }
}
```

**VS Code** (`.vscode/mcp.json`):
```json
{
  "servers": {
    "memory-os-ai": {
      "type": "stdio",
      "command": "memory-os-ai",
      "env": {
        "MEMORY_WORKSPACE": "${workspaceFolder}"
      }
    }
  }
}
```

**Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "memory-os-ai": {
      "command": "/path/to/.venv/bin/memory-os-ai",
      "args": [],
      "env": {
        "MEMORY_WORKSPACE": "~"
      }
    }
  }
}
```

---

## Configuration Reference

### Environment Variables — mcp-openclaw-extensions

| Variable | Default | Description |
|----------|---------|-------------|
| `MCP_EXT_HOST` | `127.0.0.1` | Server listen address |
| `MCP_EXT_PORT` | `8012` | Server listen port |
| `MCP_AUTH_TOKEN` | (empty) | Bearer token for auth (empty = no auth) |
| `TOOL_TIMEOUT_S` | `120` | Max seconds per tool execution |
| `LOG_LEVEL` | `INFO` | Logging level |
| `OPENCLAW_GATEWAY_URL` | `ws://127.0.0.1:18789` | Gateway WebSocket URL |
| `OPENCLAW_GATEWAY_HTTP` | `http://127.0.0.1:18789` | Gateway HTTP URL |
| `OPENCLAW_TIMEOUT_SECONDS` | `15` | Gateway request timeout |
| `VS_BRIDGE_MAX_CONTEXT_BYTES` | `32768` | Max context size for VS bridge |
| `FLEET_CONFIG_PATH` | `~/.openclaw/fleet.json` | Fleet config location |
| `FLEET_MAX_INSTANCES` | `50` | Max fleet instances |
| `FIRM_EXPORT_OUTPUT_DIR` | `~/.openclaw/exports` | Export output directory |
| `A2A_SIGNING_KEY` | (empty) | Ed25519/ES256 PEM key for JWS signing |

### Environment Variables — Memory-os-ai

| Variable | Default | Description |
|----------|---------|-------------|
| `MEMORY_WORKSPACE` | `.` | Root workspace path |
| `MEMORY_CACHE_DIR` | `~/.memory-os-ai` | FAISS index + model cache |
| `MEMORY_MODEL` | `all-MiniLM-L6-v2` | SentenceTransformer model name |
| `MEMORY_API_KEY` | (empty) | Auth key for SSE/HTTP modes |
| `TOKENIZERS_PARALLELISM` | `false` | HuggingFace tokenizer setting |

---

## Troubleshooting

### Common Issues

**"Connection refused on port 8012"**
```bash
# Check if the server is running
curl http://127.0.0.1:8012/health
# If not, start it:
cd mcp-openclaw-extensions && source .venv/bin/activate && python -m src.main
```

**"ModuleNotFoundError: aiohttp"**
```bash
# Ensure you're in the right venv
cd mcp-openclaw-extensions
source .venv/bin/activate
pip install aiohttp httpx pydantic websockets
```

**"FAISS index not found"**
```bash
# First-time setup — the index is created when you ingest your first document
memory-os-ai  # Start the server
# Then in VS Code: memory_ingest(path="some-document.pdf")
```

**"Permission denied: tesseract"**
```bash
# macOS
brew install tesseract
# Ubuntu
sudo apt install tesseract-ocr
```

**"Docker healthcheck failing"**
```bash
# Check logs
docker compose logs memory-os-ai
docker compose logs openclaw-extensions
# Verify ports aren't already in use
lsof -i :8012
lsof -i :8765
```

**"Tool timeout after 120s"**
```bash
# Increase timeout in .env
TOOL_TIMEOUT_S=300
# Restart the server
```

### Verifying the Full Stack

```bash
# 1. Health checks
curl -s http://127.0.0.1:8012/health | python -m json.tool
curl -s http://127.0.0.1:8765/health | python -m json.tool

# 2. List available tools (openclaw-extensions)
curl -s -X POST http://127.0.0.1:8012/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  | python -m json.tool | head -50

# 3. Call a tool
curl -s -X POST http://127.0.0.1:8012/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"openclaw_prompt_injection_check","arguments":{"text":"Hello world"}}}' \
  | python -m json.tool

# 4. Run the integration test suite (54 assertions)
cd setup-vs-agent-firm
python -m pytest tests/test_integration.py -v
```
