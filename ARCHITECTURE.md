# Architecture

> System architecture of the Firm AI Agent ecosystem.

⚠️ Contenu généré par IA — validation humaine requise avant utilisation.

---

## High-Level Overview

The ecosystem follows a **hub-and-spoke** architecture where VS Code acts as the hub,
connecting to two MCP servers (spokes) that each have a distinct responsibility.

```mermaid
graph TB
    subgraph IDE["VS Code (Copilot Agent Mode)"]
        Agents["Agent Files<br/>.github/agents/*.agent.md"]
        Prompts["Prompt Files<br/>.github/prompts/*.prompt.md"]
        MCPClient["MCP Client<br/>(built-in)"]
    end

    subgraph Parent["setup-vs-agent-firm"]
        Factory["factory/generate-firm.sh"]
        Souls["souls/ (9 SOUL.md)"]
        Skills["skills/ (34 SKILL.md)"]
        Docker["docker-compose.yml"]
        CI["openclaw-review.yml"]
    end

    subgraph Server1["mcp-openclaw-extensions (port 8012)"]
        Main["main.py<br/>aiohttp JSON-RPC"]
        Models["models.py<br/>138 Pydantic models"]
        Modules["29 modules<br/>138 tools"]
        Dispatch["Tool Registry<br/>+ timeout + auth"]
    end

    subgraph Server2["Memory-os-ai (port 8765)"]
        MServer["server.py<br/>MCP SDK"]
        Engine["engine.py<br/>FAISS + SentenceTransformers"]
        Chat["chat_extractor.py<br/>4 extractors"]
        MTools["18 tools"]
    end

    subgraph External["External Services"]
        Gateway["OpenClaw Gateway<br/>ws://127.0.0.1:18789"]
        ClawHub["ClawHub Registry"]
        WhatsApp["WhatsApp / Telegram / …"]
    end

    Factory -->|generates| Agents
    Factory -->|generates| Prompts
    Souls -->|identity for| Agents
    Skills -->|published to| ClawHub

    MCPClient -->|"MCP stdio"| MServer
    MCPClient -->|"MCP HTTP POST"| Main

    Main --> Dispatch
    Dispatch --> Modules
    Modules --> Models

    Main -->|"WebSocket / HTTP"| Gateway
    Gateway --> WhatsApp

    Docker -->|"orchestrates"| Server1
    Docker -->|"orchestrates"| Server2

    CI -->|"starts + calls"| Main
```

---

## Repository Boundaries

Each repository owns a specific layer of the stack:

```mermaid
graph LR
    subgraph Layer1["Layer 1: Generation & Configuration"]
        direction TB
        A1["generate-firm.sh"]
        A2["SOUL.md × 9"]
        A3["SKILL.md × 34"]
        A4["docker-compose.yml"]
        A5["mcp-config-unified.json"]
    end

    subgraph Layer2["Layer 2: Tool Execution"]
        direction TB
        B1["138 MCP tools"]
        B2["Pydantic validation"]
        B3["Bearer token auth"]
        B4["JSON-RPC 2.0 + SSE"]
        B5["Plugin: resources, prompts,<br/>elicitation, tasks"]
    end

    subgraph Layer3["Layer 3: Persistent Memory"]
        direction TB
        C1["FAISS vector index"]
        C2["Multi-format ingestion"]
        C3["Chat conversation sync"]
        C4["Cross-project linking"]
        C5["5 AI client bridges"]
    end

    Layer1 -->|"configures &<br/>deploys"| Layer2
    Layer1 -->|"configures &<br/>deploys"| Layer3
    Layer2 -.->|"complementary<br/>capabilities"| Layer3

    style Layer1 fill:#e1f5fe
    style Layer2 fill:#fff3e0
    style Layer3 fill:#e8f5e9
```

| Layer | Repository | Responsibility |
|-------|-----------|----------------|
| **Layer 1**: Generation | `setup-vs-agent-firm` | Create firms, define personas, publish skills, orchestrate deployment |
| **Layer 2**: Execution | `mcp-openclaw-extensions` | Run 138 audit/security/compliance/business tools via MCP |
| **Layer 3**: Memory | `Memory-os-ai` | Persist and retrieve semantic knowledge across sessions |

---

## Component Deep-Dives

### 1. Factory — Firm Generation Pipeline

The factory is a single Bash script that generates a complete VS Code agent
workspace from parameters:

```mermaid
flowchart LR
    Input["Input:<br/>--name, --sector,<br/>--size, --stack"]
    Factory["generate-firm.sh<br/>(846 lines)"]
    
    subgraph Output["Generated Firm"]
        direction TB
        CEO[".github/agents/<br/>firm-ceo.agent.md"]
        Dept[".github/agents/<br/>department-*.agent.md"]
        Emp[".github/agents/<br/>{dept}-{service}.agent.md"]
        Prompt[".github/prompts/<br/>firm-delivery.prompt.md"]
        Settings[".vscode/settings.json"]
        AgentsMD["AGENTS.md"]
        ClaudeMD["CLAUDE.md"]
        Install["scripts/<br/>install-skills.sh"]
    end

    Input --> Factory
    Factory --> Output
    
    CEO -->|"references"| Dept
    Dept -->|"delegates to"| Emp
    Prompt -->|"orchestrates"| CEO
```

**Sector customization** affects:
- Department structure (e.g., `legal` sector adds compliance-focused departments)
- Tool recommendations per agent
- SKILL.md installation list
- CONTRIBUTING.md templates

**Size determines department count:**

```mermaid
graph TB
    subgraph Startup["startup (4 depts)"]
        S1["Strategy"]
        S2["Engineering"]
        S3["Quality"]
        S4["Operations"]
    end

    subgraph Scaleup["scaleup (12 depts)"]
        SC1["+ Finance"]
        SC2["+ Legal"]
        SC3["+ HR"]
        SC4["+ Sales"]
        SC5["+ Marketing"]
        SC6["+ Product"]
        SC7["+ Data"]
        SC8["+ Support"]
    end

    subgraph Enterprise["enterprise (18 depts)"]
        E1["+ Security"]
        E2["+ Infrastructure"]
        E3["+ Research"]
        E4["+ Compliance"]
        E5["+ Partnerships"]
        E6["+ Design"]
    end

    Startup --> Scaleup
    Scaleup --> Enterprise
```

### 2. MCP Server — Tool Architecture

The `mcp-openclaw-extensions` server follows a modular architecture:

```mermaid
flowchart TB
    subgraph Entry["HTTP Entry Point"]
        Req["POST /mcp<br/>JSON-RPC 2.0"]
        Auth["Bearer Token<br/>Auth (optional)"]
        SSE["GET /mcp/sse<br/>Server-Sent Events"]
    end

    subgraph Dispatch["Dispatcher"]
        Parse["Parse JSON-RPC"]
        Route["Route by method:<br/>tools/call, tools/list,<br/>resources/read, prompts/get"]
        Validate["Pydantic Model<br/>Validation"]
        Timeout["asyncio.wait_for<br/>(120s default)"]
    end

    subgraph Modules["29 Modules (138 tools)"]
        direction TB
        M1["Security (45 tools)<br/>security_audit, advanced_security,<br/>prompt_security, gateway_hardening,<br/>runtime_audit"]
        M2["Infrastructure (28 tools)<br/>gateway_fleet, config_migration,<br/>acp_bridge, reliability_probe"]
        M3["Protocol (16 tools)<br/>a2a_bridge, n8n_bridge,<br/>vs_bridge, spec_compliance"]
        M4["Compliance (15 tools)<br/>auth_compliance, compliance_medium,<br/>platform_audit"]
        M5["Memory & Analytics (15 tools)<br/>hebbian_memory, observability,<br/>memory_audit, ecosystem_audit"]
        M6["Business (19 tools)<br/>legal_status, location_strategy,<br/>market_research, supplier_mgmt,<br/>delivery_export"]
    end

    subgraph Shared["Shared Infrastructure"]
        ConfigHelpers["config_helpers.py<br/>load_config, get_nested,<br/>mask_secret, no_traversal"]
        ModelsFile["models.py<br/>138 Pydantic models<br/>TOOL_MODELS registry"]
    end

    Req --> Auth --> Parse
    SSE --> Parse
    Parse --> Route
    Route --> Validate
    Validate --> Timeout
    Timeout --> Modules

    Modules --> ConfigHelpers
    Modules --> ModelsFile
```

**Tool registration pattern** — each module exports a `TOOLS` list:

```python
# Example: security_audit.py
TOOLS = [
    {
        "name": "openclaw_security_scan",
        "title": "Security Scan",
        "description": "Scan OpenClaw config for SQL injection...",
        "inputSchema": { ... },
        "annotations": {"audience": ["operator"], "readOnlyHint": True},
    },
    # ... more tools
]

async def handle_openclaw_security_scan(params: dict) -> dict:
    validated = SecurityScanInput(**params)  # Pydantic validation
    # ... implementation
    return {"ok": True, "findings": [...], "severity": "clean"}
```

### 3. Memory Engine — FAISS Pipeline

```mermaid
flowchart TB
    subgraph Ingestion["Document Ingestion"]
        PDF["PDF<br/>(pymupdf + OCR)"]
        DOCX["DOCX<br/>(python-docx)"]
        PPTX["PPTX<br/>(python-pptx)"]
        IMG["Images<br/>(Pillow + tesseract)"]
        Audio["Audio<br/>(Whisper)"]
    end

    subgraph Processing["Processing Pipeline"]
        Chunk["Text Chunking<br/>(512 tokens)"]
        Embed["SentenceTransformers<br/>all-MiniLM-L6-v2"]
        Index["FAISS FlatL2<br/>Vector Index"]
    end

    subgraph ChatSync["Chat Synchronization"]
        VSCode["VS Code Copilot<br/>(SQLite DB)"]
        JSONL["JSONL Logs"]
        Markdown["Markdown Exports"]
        Folder["Folder Watch"]
    end

    subgraph Query["Query Interface"]
        Search["memory_search<br/>(similarity)"]
        Context["memory_get_context<br/>(window around match)"]
        Brief["memory_session_brief<br/>(recent activity summary)"]
    end

    PDF --> Chunk
    DOCX --> Chunk
    PPTX --> Chunk
    IMG --> Chunk
    Audio --> Chunk

    Chunk --> Embed
    Embed --> Index

    VSCode --> Chunk
    JSONL --> Chunk
    Markdown --> Chunk
    Folder --> Chunk

    Index --> Search
    Index --> Context
    Index --> Brief
```

---

## Deployment Architecture

### Docker Compose

Both MCP servers run as containers orchestrated by `docker-compose.yml` in
`setup-vs-agent-firm`:

```mermaid
graph TB
    subgraph DockerHost["Docker Host"]
        subgraph Network["Docker Network (default)"]
            subgraph C1["Container: memory-os-ai"]
                MOS["Memory-os-ai<br/>Python 3.11-slim"]
                FAISS["FAISS Index<br/>/data/.cache"]
                P1["Port 8765<br/>Streamable HTTP"]
            end

            subgraph C2["Container: openclaw-extensions"]
                OCE["mcp-openclaw-ext<br/>Python 3.11-slim"]
                ENV[".env config"]
                P2["Port 8012<br/>HTTP"]
            end
        end

        V1["Volume:<br/>memory-data"]
        V2["Bind mount:<br/>./workspace"]
    end

    VS["VS Code<br/>(host)"] -->|"localhost:8012"| P2
    VS -->|"localhost:8765"| P1
    V1 --> FAISS
    V2 --> C1
    V2 --> C2
```

### VS Code MCP Configuration

The `mcp-config-unified.json` connects both servers:

```json
{
  "servers": {
    "memory-os-ai": {
      "type": "stdio",
      "command": "memory-os-ai"
    },
    "openclaw-extensions": {
      "type": "http",
      "url": "http://127.0.0.1:8012/mcp"
    }
  }
}
```

When running locally (not Docker), Memory-os-ai uses **stdio** transport
(direct process communication), while openclaw-extensions always uses **HTTP**.

---

## Security Architecture

```mermaid
flowchart TB
    subgraph Guards["Input Guards"]
        PT["Path Traversal<br/>Block .. in all paths"]
        PV["Pydantic Validation<br/>Type + constraints"]
        RL["Request Size Limit<br/>2MB max"]
        TO["Tool Timeout<br/>120s default"]
    end

    subgraph Auth["Authentication"]
        Bearer["Bearer Token<br/>(MCP_AUTH_TOKEN)"]
        HMAC["Timing-safe<br/>hmac.compare_digest"]
        MemKey["MEMORY_API_KEY<br/>(optional)"]
    end

    subgraph Audit["Security Audit Tools"]
        Scan["security_scan"]
        Inject["prompt_injection_check<br/>(16 regex patterns)"]
        Secrets["secrets_lifecycle_check"]
        Proto["config_prototype_check"]
        Sandbox["sandbox_audit"]
    end

    subgraph Output["Output Safety"]
        Mask["_mask_secret()<br/>(last 4 chars only)"]
        NoLog["No tokens in logs"]
        GitIgnore[".env in .gitignore"]
    end

    Guards --> Auth
    Auth --> Audit
    Audit --> Output
```

**Key security patterns:**
- **`_no_traversal(path)`**: Blocks `..` in every file path parameter (via `ConfigPathInput` base class)
- **16 compiled regex patterns** for prompt injection detection (CRITICAL: override/ChatML, HIGH: DAN/jailbreak, MEDIUM: base64/exfiltration)
- **SSRF protection**: localhost/127.0.0.1/0.0.0.0/::1 blocked in A2A bridge
- **Advisory file locks**: `fcntl.LOCK_EX | LOCK_NB` for workspace locking

---

## CI/CD Architecture

```mermaid
flowchart LR
    subgraph Triggers["PR Triggers"]
        PR1["PR → setup-vs-agent-firm"]
        PR2["PR → mcp-openclaw"]
        PR3["PR → Memory-os-ai"]
    end

    subgraph CI1["openclaw-review.yml"]
        Diff["Collect PR diff"]
        Start["Start MCP server"]
        Review["AI Quality Review"]
        Comment["Post PR comment"]
    end

    subgraph CI2["mcp-openclaw CI"]
        Lint2["Ruff lint"]
        Test2["pytest + coverage<br/>(fail-under 94%)"]
        Sec2["TruffleHog<br/>secrets scan"]
    end

    subgraph CI3["Memory-os-ai CI"]
        Matrix["Matrix 3.10/3.11/3.12"]
        Lint3["Ruff lint"]
        Test3["pytest + coverage<br/>(fail-under 80%)"]
        Sec3["TruffleHog<br/>secrets scan"]
    end

    PR1 --> CI1
    PR2 --> CI2
    PR3 --> CI3

    CI2 --> |"required for merge"| Merge2["✅ Merge"]
    CI3 --> |"required for merge"| Merge3["✅ Merge"]
```

**Branch protection** is enforced on all 3 repositories:
- Force push: blocked
- Branch delete: blocked
- Required checks: `test` (mcp-openclaw), `test (3.11)` (Memory-os-ai)
- Admin enforcement: enabled

---

## Technology Stack Summary

```mermaid
mindmap
  root((Firm Ecosystem))
    Languages
      Python 3.11
      Bash
      YAML / JSON
      Markdown
    Frameworks
      aiohttp (HTTP server)
      MCP SDK (memory server)
      Pydantic v2 (validation)
      FAISS (vectors)
      SentenceTransformers
    Infrastructure
      Docker Compose
      GitHub Actions
      VS Code Copilot
      OpenClaw Gateway
    Protocols
      MCP 2025-11-25
      A2A Protocol v1.0 RC
      JSON-RPC 2.0
      Server-Sent Events
      WebSocket
    Quality
      pytest (2,931 tests)
      Ruff (linting)
      TruffleHog (secrets)
      94-96% coverage
```
