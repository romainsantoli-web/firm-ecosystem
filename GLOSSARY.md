# Glossary

> Terminology reference for the Firm AI Agent ecosystem.

⚠️ Contenu généré par IA — validation humaine requise avant utilisation.

---

## A

### A2A Protocol
**Agent-to-Agent Protocol** — An open protocol (RC v1.0) enabling AI agents to discover each
other's capabilities, send tasks, receive results, subscribe to status updates, and cancel
operations. The ecosystem implements this via `a2a_bridge.py` (8 tools).

### ACP
**Agent Communication Protocol** — Protocol for persistence, scheduling, and locking of
agent sessions. Implemented via `acp_bridge.py` (7 tools) with JSON file persistence and
`fcntl` advisory locks.

### ADR
**Architecture Decision Record** — Lightweight document recording a significant
architectural decision, its context, and consequences. The ecosystem uses MADR format.
Generated via `firm_adr_generate` tool.

### Agent Card
JSON document describing an A2A agent's identity, capabilities, skills, auth requirements,
and supported input/output modes. Generated from SOUL.md files via `openclaw_a2a_card_generate`.

### Agent Mode
VS Code Copilot feature that enables AI agents defined in `.github/agents/` to be invoked
in the chat panel. Agents can call MCP tools and delegate to each other.

---

## B

### Bearer Token
HTTP authentication scheme used by `mcp-openclaw-extensions`. When `MCP_AUTH_TOKEN` is set,
all requests must include `Authorization: Bearer <token>`. Validated with
`hmac.compare_digest` for timing-safety.

---

## C

### ClawHub
Public registry for SKILL.md packs (analogous to npm for OpenClaw skills). The ecosystem
publishes 34 skill packs. CLI: `npx clawhub@latest install <skill-name>`.

### Claude Code
Anthropic's AI coding assistant with MCP client support. Memory-os-ai auto-configures
via `memory-os-ai setup claude-code`, generating `.claude/mcp.json`.

### Co-activation
In Hebbian memory: two tools called within the same session window are co-activated.
Repeated co-activation strengthens their connection weight, enabling pattern discovery.

### ConfigPathInput
Base Pydantic model class in `models.py` providing path-traversal guard (`_no_traversal`)
and `max_length` constraint. All tools accepting file paths inherit from this.

---

## D

### Department
An organizational unit within a Firm. Each department has a head agent and multiple
service agents. Sizes: startup (4), scaleup (12), enterprise (18).

### Delivery Export
Pipeline for publishing agent deliverables to external platforms. 6 tools:
GitHub PR, Jira ticket, Linear issue, Slack digest, Document export, Auto-detect.

---

## E

### Elicitation
MCP capability (2025-11-25 spec) allowing a server to request additional information
from the user during tool execution. Enabled in `mcp-openclaw-extensions`.

---

## F

### Factory
The `generate-firm.sh` script (846 lines) that generates a complete VS Code agent
firm from parameters (name, sector, size, stack).

### FAISS
**Facebook AI Similarity Search** — Library for efficient nearest-neighbor search in
high-dimensional vector spaces. Used by Memory-os-ai for the semantic index (FlatL2).

### Firm
A virtual organization of AI agents structured as a corporate hierarchy. Contains a CEO
agent, department heads, and specialist service agents.

### Fleet
Multiple instances of OpenClaw Gateway running simultaneously. Managed via 6 tools:
`firm_gateway_fleet_{status,add,remove,broadcast,sync,list}`.

---

## G

### Gateway
**OpenClaw Gateway** — The central server that connects AI agents to messaging channels
(WhatsApp, Telegram, Discord, etc.). Runs on port 18789 (WebSocket).

---

## H

### Hebbian Memory
Adaptive memory module inspired by Hebb's rule ("neurons that fire together wire together").
8 tools track tool co-activation patterns, adjust weights, detect drift, and strip PII.
Split across 5 submodules in `src/hebbian_memory/`.

### Healthcheck
HTTP endpoint (`/health`) returning server status. Both MCP servers expose this.
Docker uses it for container health monitoring.

---

## J

### JSON-RPC 2.0
The wire protocol used by MCP. Requests are `{"jsonrpc":"2.0","id":N,"method":"...","params":{...}}`.
The `mcp-openclaw-extensions` server implements this over HTTP POST on `/mcp`.

### JWS
**JSON Web Signature** — Used for signing A2A Agent Cards. Supports Ed25519 and ES256
algorithms. Key configured via `A2A_SIGNING_KEY` environment variable.

---

## M

### MCP
**Model Context Protocol** — Open protocol for connecting AI models to external tools
and data sources. Supports stdio, SSE, and Streamable HTTP transports.
Current version: 2025-11-25.

### MCP Client
The component in VS Code (or other AI clients) that connects to MCP servers and
routes tool calls. VS Code's built-in MCP client is configured via `.vscode/mcp.json`.

### MCP Server
A server implementing the MCP protocol, exposing tools, resources, and prompts.
The ecosystem has two: `mcp-openclaw-extensions` (138 tools) and Memory-os-ai (18 tools).

---

## N

### n8n
Open-source workflow automation tool. The `n8n_bridge.py` module (2 tools) converts
between OpenClaw pipelines and n8n workflow JSON format.

---

## O

### OpenClaw
Open-source platform for deploying AI agents with multi-channel messaging support
(WhatsApp, Telegram, Discord, Slack, etc.). The Firm ecosystem extends it with
138 MCP tools.

---

## P

### Prompt
MCP capability for serving parameterized prompt templates. The ecosystem exposes 5:
`security-audit`, `a2a-card-review`, `gap-analysis`, `config-review`,
`tool-deprecation-plan`.

### Pydantic
Python data validation library (v2). Every tool input is validated via a Pydantic model
in `models.py`. The `TOOL_MODELS` dict maps tool names to model classes.

### Pyramid
The hierarchical structure of a Firm: CEO → Department Heads → Service Agents.
All communication flows through the hierarchy.

---

## R

### Resource
MCP capability for exposing read-only data URIs. `mcp-openclaw-extensions` exposes 2:
`config://openclaw/main` and `audit://openclaw/last-run`.
Memory-os-ai exposes 3: `memory://documents/*`, `memory://logs/conversation`,
`memory://linked/*`.

---

## S

### SentenceTransformers
Python library for computing text embeddings. Memory-os-ai uses the
`all-MiniLM-L6-v2` model (384-dimensional vectors) by default.

### SKILL.md
A skill definition file publishable to ClawHub. Contains YAML frontmatter
(name, version, registry metadata) + documentation + tool activation instructions.
34 packs in the ecosystem.

### SOUL.md
A persona definition file for AI agents. Contains YAML frontmatter + identity +
core values + communication style + decision framework. 9 souls in the ecosystem.

### SSE
**Server-Sent Events** — Unidirectional streaming from server to client over HTTP.
Used by MCP for streaming tool results (`GET /mcp/sse`) and by A2A for task
subscription (`subscribe` tool).

### SSRF
**Server-Side Request Forgery** — Attack where a server is tricked into making
requests to internal services. The A2A bridge blocks localhost/127.0.0.1/0.0.0.0/::1.

---

## T

### Tasks (MCP)
MCP capability (2025-11-25 spec) for durable, long-running operations that survive
connection interrupts. Enabled in `mcp-openclaw-extensions`.

### Tool
An MCP tool is a callable function exposed by a server. Each tool has a name,
description, input schema (JSON Schema), and optional annotations.
Total across the ecosystem: 156 (138 + 18).

### Tool Registry
The `TOOL_REGISTRY` dict in `main.py` that maps tool names to their handler functions.
Populated at startup from all 29 modules.

---

## V

### VS Bridge
4 tools that synchronize context between VS Code and the OpenClaw Gateway:
`vs_context_push`, `vs_context_pull`, `vs_session_link`, `vs_session_status`.

---

## W

### Workspace Lock
Advisory file lock (`fcntl.LOCK_EX | LOCK_NB`) preventing concurrent modifications
to a workspace. Implemented via `openclaw_workspace_lock` tool.
