# Firm Ecosystem

> Architecture documentation for the **Firm AI Agent** ecosystem — three repositories
> working together to create, deploy, and operate autonomous AI agent "firms" on
> [OpenClaw](https://github.com/open-claw) + [ClawHub](https://clawhub.com).

⚠️ Contenu généré par IA — validation humaine requise avant utilisation.

---

## 🎯 What is a "Firm"?

A **Firm** is a virtual organization of AI agents structured as a corporate hierarchy.
Each firm has a CEO, departments, and specialized employees — all running as
[VS Code Copilot agents](https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode)
connected to an [OpenClaw Gateway](https://github.com/open-claw) for orchestration.

```
          ┌─────────────────┐
          │    CEO Agent     │  ← Strategic orchestrator
          │ (Alexandra M.)  │
          └───────┬─────────┘
    ┌─────────────┼─────────────┐
    ▼             ▼             ▼
┌───────┐   ┌───────┐   ┌───────┐
│ Dept  │   │ Dept  │   │ Dept  │   ... up to 18 departments
│ Head  │   │ Head  │   │ Head  │
└───┬───┘   └───┬───┘   └───┬───┘
    ▼           ▼           ▼
┌───────┐ ┌───────┐   ┌───────┐
│Service│ │Service│   │Service│  ← Specialist agents
│ Agent │ │ Agent │   │ Agent │
└───────┘ └───────┘   └───────┘
```

**Three sizes** of firms are supported:

| Size | Departments | Use Case |
|------|-------------|----------|
| **Startup** | 4 (strategy, engineering, quality, ops) | Small projects, rapid prototyping |
| **Scaleup** | 12 | Growing companies, multi-team work |
| **Enterprise** | 18 (all departments) | Large-scale operations, full coverage |

**Fifteen industry sectors** are pre-configured: generic, legal, medtech, ecommerce,
fintech, saas, manufacturing, education, real estate, logistics, media, automotive,
energy, HR, consulting.

---

## 🏗️ The Three Repositories

The ecosystem is composed of three repositories, each with a distinct role:

| Repository | Role | Language | Version | License |
|------------|------|----------|---------|---------|
| [**setup-vs-agent-firm**](https://github.com/romainsantoli-web/setup-vs-agent-firm) | Orchestrator & Factory | Bash + YAML | — | — |
| [**mcp-openclaw-extensions**](https://github.com/romainsantoli-web/mcp-openclaw) | MCP Tool Server | Python 3.11 | v3.3.0 | MIT |
| [**Memory-os-ai**](https://github.com/romainsantoli-web/Memory-os-ai) | Semantic Memory Engine | Python 3.11 | v3.0.0 | LGPL-3.0 |

### 1. `setup-vs-agent-firm` — The Orchestrator

The **parent repository** that ties everything together. It contains:

- **`factory/generate-firm.sh`** — An 846-line Bash script that generates a complete
  VS Code agent firm (agents, prompts, settings, scripts) for any sector/size/stack
  combination
- **`souls/`** — 9 SOUL.md persona files defining agent identities (CEO, CTO, CFO,
  Legal, HR, Location, Market Research, Suppliers, Legal Status)
- **`skills/`** — 34 SKILL.md packs published on ClawHub, activating specific tool
  groups
- **`docker-compose.yml`** — Orchestrates both MCP servers as Docker containers
- **`.github/workflows/openclaw-review.yml`** — AI-powered PR review using the MCP tools
- **`mcp-config-unified.json`** — VS Code MCP configuration for both servers

### 2. `mcp-openclaw-extensions` — The Tool Server

A Python **MCP server** exposing **138 tools** across **29 modules** on port 8012.
These tools cover:

| Category | Tools | Examples |
|----------|-------|---------|
| Security & Audit | 45 | Injection detection, sandbox audit, secrets lifecycle |
| Infrastructure | 28 | Gateway auth, fleet management, config migration |
| Protocol Bridges | 16 | A2A Protocol, n8n workflows, ACP persistence |
| Compliance | 15 | MCP spec, OAuth/OIDC, GDPR, deprecation audit |
| Platform | 15 | Observability, i18n, knowledge graph, browser audit |
| Business | 19 | Legal status, market research, supplier management |

Full API reference: [API-REFERENCE.md](API-REFERENCE.md)

### 3. `Memory-os-ai` — The Memory Engine

A universal **semantic memory server** exposing **18 tools** via MCP. It provides:

- **FAISS vector index** for similarity search across all ingested documents
- **Multi-format ingestion**: PDF, DOCX, PPTX, images (OCR), audio (Whisper)
- **Chat sync**: Extracts and indexes conversations from VS Code Copilot, Claude,
  ChatGPT
- **Cross-project linking**: Share memory between multiple workspaces
- **5 AI client bridges**: Claude Code, Claude Desktop, Codex CLI, VS Code, ChatGPT

---

## 🔌 How They Connect

```
┌─────────────────────────────────────────────────────────┐
│                      VS Code IDE                         │
│                                                          │
│   Copilot Agent Mode     +     MCP Client                │
│   ├── .github/agents/    │     ├── memory-os-ai (stdio)  │
│   ├── .github/prompts/   │     └── openclaw-ext (HTTP)   │
│   └── AGENTS.md          │                               │
└──────────────┬───────────┴──────────┬────────────────────┘
               │                      │
          MCP stdio              MCP HTTP POST
               │                      │
    ┌──────────▼──────────┐  ┌────────▼──────────────┐
    │   Memory-os-ai      │  │ mcp-openclaw-extensions│
    │   Port 8765          │  │  Port 8012             │
    │                      │  │                        │
    │  18 tools            │  │  138 tools             │
    │  FAISS vector index  │  │  29 modules            │
    │  Chat sync engine    │  │  Pydantic validation   │
    │  Document ingestion  │  │  Bearer token auth     │
    └──────────────────────┘  └────────┬───────────────┘
                                       │
                                  WebSocket / HTTP
                                       │
                              ┌────────▼───────────┐
                              │  OpenClaw Gateway   │
                              │  Port 18789         │
                              │                     │
                              │  14-department       │
                              │  agent pyramid       │
                              │  WhatsApp/Telegram/…│
                              └─────────────────────┘
```

For detailed data flow diagrams, see [DATA-FLOW.md](DATA-FLOW.md).

For system architecture deep-dive, see [ARCHITECTURE.md](ARCHITECTURE.md).

---

## 🚀 Quick Start

### Option A: Docker Compose (recommended)

```bash
# 1. Clone the parent repo
git clone https://github.com/romainsantoli-web/setup-vs-agent-firm.git
cd setup-vs-agent-firm

# 2. Clone Memory-os-ai next to it
cd ..
git clone https://github.com/romainsantoli-web/Memory-os-ai.git
cd setup-vs-agent-firm

# 3. Configure environment
cp mcp-openclaw-extensions/.env.example mcp-openclaw-extensions/.env
# Edit .env with your API keys

# 4. Start both servers
docker compose up -d

# 5. Verify
curl http://127.0.0.1:8012/health   # openclaw-extensions
curl http://127.0.0.1:8765/health   # memory-os-ai
```

### Option B: Local Development

```bash
# 1. Clone repos
git clone https://github.com/romainsantoli-web/setup-vs-agent-firm.git
git clone https://github.com/romainsantoli-web/Memory-os-ai.git

# 2. Setup mcp-openclaw-extensions
cd setup-vs-agent-firm/mcp-openclaw-extensions
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
python -m src.main  # Starts on port 8012

# 3. Setup Memory-os-ai (separate terminal)
cd Memory-os-ai
python -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"
memory-os-ai  # Starts in stdio mode (for Claude Code / VS Code)
# or: memory-os-ai --sse   # SSE on port 8765
# or: memory-os-ai --http  # Streamable HTTP on port 8765
```

### Option C: Generate a Firm

```bash
cd setup-vs-agent-firm

# Generate a fintech startup firm with TypeScript stack
bash factory/generate-firm.sh \
  --name "MyFintechFirm" \
  --sector fintech \
  --size startup \
  --stack typescript \
  --output ~/projects/my-firm

# This creates:
# ~/projects/my-firm/.github/agents/    — Agent definitions
# ~/projects/my-firm/.github/prompts/   — Orchestration prompts
# ~/projects/my-firm/.vscode/settings.json
# ~/projects/my-firm/AGENTS.md
# ~/projects/my-firm/CLAUDE.md
# ~/projects/my-firm/CONTRIBUTING.md
# ~/projects/my-firm/scripts/install-skills.sh
```

Full setup guide with prerequisites: [SETUP-GUIDE.md](SETUP-GUIDE.md)

---

## 📖 Documentation Index

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture, component interactions, Mermaid diagrams |
| [DATA-FLOW.md](DATA-FLOW.md) | Request lifecycle, data flow between components, sequence diagrams |
| [SETUP-GUIDE.md](SETUP-GUIDE.md) | Detailed installation, prerequisites, configuration reference |
| [API-REFERENCE.md](API-REFERENCE.md) | All 156 MCP tools (138 + 18) with parameters and examples |
| [EXAMPLES.md](EXAMPLES.md) | End-to-end use case walkthroughs with code samples |
| [GLOSSARY.md](GLOSSARY.md) | Terminology reference for the Firm ecosystem |
| [decisions/](decisions/) | Architecture Decision Records (ADRs) |

---

## 🏭 Products Built with Firm

These real applications were built end-to-end using the Firm CEO orchestration pipeline:

| Product | Stack | Repo | Status |
|---------|-------|------|--------|
| [**TaskFlow Pro**](https://github.com/romainsantoli-web/taskflow-pro) | Express + TypeScript + React + SQLite | `romainsantoli-web/taskflow-pro` | ✅ v1.0.0 |

**TaskFlow Pro** — Professional SaaS project management (Kanban, analytics, team collaboration).
Built using `firm init --sector saas --size startup --stack typescript` + CEO delegation protocol
(Strategy → Engineering → Quality → Operations). 23 files, 2962 LoC, 26 tests, Docker + CI/CD.

---

## 📊 Ecosystem Stats

| Metric | Value |
|--------|-------|
| **Total MCP tools** | 156 (138 + 18) |
| **Total tests** | 2,931 (2,583 + 348) |
| **Test coverage** | 94% / 96% |
| **SOUL.md personas** | 9 |
| **SKILL.md packs** | 34 |
| **Industry sectors** | 15 |
| **Products built** | 1 (TaskFlow Pro) |
| **Supported AI clients** | 5 (Claude Code, Claude Desktop, Codex, VS Code, ChatGPT) |
| **Supported transports** | stdio, SSE, Streamable HTTP |
| **MCP Protocol version** | 2025-11-25 |

---

## 🤝 Contributing

Each repository has its own contribution guide:

- [setup-vs-agent-firm CONTRIBUTING](https://github.com/romainsantoli-web/setup-vs-agent-firm/blob/main/CONTRIBUTING.md)
- [mcp-openclaw-extensions](https://github.com/romainsantoli-web/mcp-openclaw) — See README
- [Memory-os-ai](https://github.com/romainsantoli-web/Memory-os-ai) — See README

### Common Rules

1. **Git**: Always work on a feature branch (`feat/`, `fix/`, `chore/`)
2. **Pydantic**: All tool inputs validated via Pydantic v2 models with path-traversal guards
3. **Tests**: `pytest -v` must pass at 100% before push; coverage ≥ 80%
4. **Secrets**: Never commit tokens, keys, or passwords
5. **AI output**: Mark generated deliverables with `⚠️ Contenu généré par IA`

---

## 📄 License

- **setup-vs-agent-firm**: See repository
- **mcp-openclaw-extensions**: MIT
- **Memory-os-ai**: LGPL-3.0-or-later
- **This documentation**: MIT
