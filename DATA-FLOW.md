# Data Flow

> Request lifecycle, data flow patterns, and sequence diagrams for the Firm ecosystem.

⚠️ Contenu généré par IA — validation humaine requise avant utilisation.

---

## 1. Tool Call Lifecycle

When a VS Code Copilot agent invokes an MCP tool, the request flows through
multiple layers of validation and processing:

```mermaid
sequenceDiagram
    participant User as Developer
    participant VS as VS Code Copilot
    participant MCP as mcp-openclaw-extensions
    participant Pydantic as Pydantic Validation
    participant Handler as Tool Handler
    participant GW as OpenClaw Gateway

    User->>VS: Ask agent to perform task
    VS->>VS: Agent selects appropriate tool
    VS->>MCP: POST /mcp (JSON-RPC 2.0)<br/>{"method":"tools/call","params":{"name":"...","arguments":{...}}}

    alt Auth enabled
        MCP->>MCP: Verify Bearer token<br/>(hmac.compare_digest)
    end

    MCP->>MCP: Parse JSON-RPC envelope
    MCP->>Pydantic: Validate arguments<br/>via TOOL_MODELS[tool_name]

    alt Validation fails
        Pydantic-->>MCP: ValidationError
        MCP-->>VS: JSON-RPC error response
        VS-->>User: "Invalid parameters: ..."
    end

    Pydantic-->>MCP: Validated model instance
    MCP->>Handler: await handler(params)<br/>with asyncio.wait_for(120s)

    alt Tool needs Gateway
        Handler->>GW: WebSocket / HTTP request
        GW-->>Handler: Response
    end

    Handler-->>MCP: Result dict
    MCP-->>VS: JSON-RPC success response<br/>{"result":{"content":[{"type":"text","text":"..."}]}}
    VS-->>User: Display formatted result
```

### Error Handling

```mermaid
flowchart TB
    Request["Incoming Request"] --> SizeCheck{"Size ≤ 2MB?"}
    SizeCheck -->|No| Reject413["413 Too Large"]
    SizeCheck -->|Yes| AuthCheck{"Auth required?"}
    AuthCheck -->|No| Parse
    AuthCheck -->|Yes| TokenOK{"Token valid?"}
    TokenOK -->|No| Reject401["401 Unauthorized"]
    TokenOK -->|Yes| Parse["Parse JSON-RPC"]
    Parse --> Method{"Method?"}
    Method -->|tools/call| FindTool{"Tool exists?"}
    Method -->|tools/list| ReturnList["Return 138 tools"]
    Method -->|resources/read| ReadResource["Read resource"]
    Method -->|prompts/get| GetPrompt["Return prompt"]
    Method -->|unknown| Reject404["Method not found"]
    FindTool -->|No| RejectTool["Tool not found error"]
    FindTool -->|Yes| Validate["Pydantic validate"]
    Validate -->|Fail| RejectValidation["Validation error"]
    Validate -->|Pass| Execute["Execute with timeout"]
    Execute -->|Timeout| Reject504["Timeout error"]
    Execute -->|Exception| Reject500["Internal error"]
    Execute -->|Success| Return200["Success response"]
```

---

## 2. Memory Ingestion Flow

When a document is ingested into Memory-os-ai, it passes through a processing
pipeline:

```mermaid
sequenceDiagram
    participant User as Developer
    participant VS as VS Code
    participant Mem as Memory-os-ai
    participant Proc as Document Processor
    participant ST as SentenceTransformers
    participant FAISS as FAISS Index

    User->>VS: "Ingest this PDF into memory"
    VS->>Mem: memory_ingest(path="/docs/spec.pdf")

    Mem->>Proc: Detect format (PDF)
    Proc->>Proc: Extract text (pymupdf)

    alt OCR needed
        Proc->>Proc: pytesseract OCR fallback
    end

    Proc-->>Mem: Raw text (string)
    Mem->>Mem: Chunk text (512 tokens,<br/>50 token overlap)

    loop For each chunk
        Mem->>ST: Encode chunk
        ST-->>Mem: Embedding vector (384d)
        Mem->>FAISS: Add vector to index
    end

    Mem->>Mem: Save index to disk<br/>(~/.memory-os-ai/)
    Mem-->>VS: {"ok": true, "chunks": 47, "document_id": "..."}
    VS-->>User: "Ingested spec.pdf (47 chunks)"
```

### Supported Formats

```mermaid
graph LR
    subgraph Input["Input Formats"]
        PDF["📄 PDF"]
        DOCX["📝 DOCX"]
        PPTX["📊 PPTX"]
        IMG["🖼️ Images<br/>(PNG, JPG, TIFF)"]
        AUDIO["🎵 Audio<br/>(MP3, WAV, M4A)"]
        DOC["📃 DOC (legacy)"]
    end

    subgraph Extractors["Text Extractors"]
        PyMuPDF["pymupdf"]
        PythonDocx["python-docx"]
        PythonPptx["python-pptx"]
        Tesseract["pytesseract<br/>(OCR)"]
        Whisper["openai-whisper"]
        Textract["textract/antiword"]
    end

    subgraph Pipeline["Pipeline"]
        Chunk["Chunking<br/>(512 tokens)"]
        Embed["Embedding<br/>(all-MiniLM-L6-v2)"]
        Store["FAISS<br/>FlatL2 Index"]
    end

    PDF --> PyMuPDF --> Chunk
    DOCX --> PythonDocx --> Chunk
    PPTX --> PythonPptx --> Chunk
    IMG --> Tesseract --> Chunk
    AUDIO --> Whisper --> Chunk
    DOC --> Textract --> Chunk

    Chunk --> Embed --> Store
```

---

## 3. Chat Synchronization Flow

Memory-os-ai can automatically extract and index conversations from multiple
AI clients:

```mermaid
sequenceDiagram
    participant Sources as Chat Sources
    participant Extractor as Chat Extractor
    participant State as Sync State
    participant Engine as Memory Engine

    Note over Sources: VS Code Copilot (SQLite)<br/>JSONL Logs<br/>Markdown Exports<br/>Folder Watch

    Sources->>Extractor: memory_chat_sync(source_type="vscode")

    Extractor->>State: Load _chat_sync_state.json<br/>(last byte offset, file hashes)

    Extractor->>Sources: Read new entries<br/>(only since last sync)

    loop For each new conversation
        Extractor->>Extractor: Parse human/assistant turns
        Extractor->>Engine: Ingest conversation text
        Engine->>Engine: Chunk → Embed → Index
    end

    Extractor->>State: Save updated state<br/>(atomic write via tempfile + os.replace)

    Extractor-->>Sources: {"synced": 12, "skipped": 3, "total_chunks": 156}
```

### Chat Source Paths

| Source | Location | Format |
|--------|----------|--------|
| VS Code Copilot | `~/Library/Application Support/Code/User/globalStorage/github.copilot-chat/` | SQLite |
| Claude Code | `.claude/` or `CLAUDE.md` | JSONL / Markdown |
| ChatGPT | Export folder | JSON |
| Generic | Any folder | Markdown, JSONL, text |

---

## 4. Firm Generation Flow

When the factory generates a new firm, it produces a complete VS Code workspace:

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Factory as generate-firm.sh
    participant FS as File System
    participant ClawHub as ClawHub Registry

    Dev->>Factory: bash generate-firm.sh<br/>--name MyFirm --sector fintech<br/>--size startup --stack typescript

    Factory->>Factory: Validate parameters
    Factory->>Factory: Select departments (4 for startup)
    Factory->>Factory: Map sector → custom agents

    Factory->>FS: Create .github/agents/firm-ceo.agent.md
    Factory->>FS: Create .github/agents/department-strategy.agent.md
    Factory->>FS: Create .github/agents/department-engineering.agent.md
    Factory->>FS: Create .github/agents/department-quality.agent.md
    Factory->>FS: Create .github/agents/department-operations.agent.md

    loop For each department
        Factory->>FS: Create service agent files<br/>(2-5 per department)
    end

    Factory->>FS: Create .github/prompts/firm-delivery.prompt.md
    Factory->>FS: Create .vscode/settings.json
    Factory->>FS: Create AGENTS.md (routing map)
    Factory->>FS: Create CLAUDE.md (rules)
    Factory->>FS: Create CONTRIBUTING.md
    Factory->>FS: Create scripts/install-skills.sh

    Dev->>FS: cd MyFirm && bash scripts/install-skills.sh
    FS->>ClawHub: npx clawhub install firm-orchestration
    FS->>ClawHub: npx clawhub install firm-fintech-pack
    FS->>ClawHub: npx clawhub install firm-security-audit
    ClawHub-->>FS: Skills installed

    Dev->>Dev: Open in VS Code → Agents active
```

---

## 5. Security Audit Flow

A typical security audit uses multiple tools in sequence:

```mermaid
sequenceDiagram
    participant Agent as Security Agent
    participant MCP as mcp-openclaw-extensions
    participant Config as OpenClaw Config

    Agent->>MCP: openclaw_security_scan<br/>(config_path="/etc/openclaw/config.yml")
    MCP->>Config: Parse YAML config
    Config-->>MCP: Config object
    MCP->>MCP: Check SQL injection patterns
    MCP->>MCP: Check path traversal vectors
    MCP-->>Agent: {findings: [...], severity: "high"}

    Agent->>MCP: openclaw_prompt_injection_check<br/>(text="user input to analyze")
    MCP->>MCP: Run 16 compiled regex patterns<br/>(override, ChatML, DAN, base64...)
    MCP-->>Agent: {injections_found: 2, severity: "critical"}

    Agent->>MCP: openclaw_secrets_lifecycle_check<br/>(config_path="...")
    MCP->>Config: Scan for hardcoded secrets
    MCP-->>Agent: {findings: [...], severity: "medium"}

    Agent->>MCP: openclaw_gateway_auth_check<br/>(config_path="...")
    MCP->>Config: Verify auth configuration
    MCP-->>Agent: {ok: true, severity: "clean"}

    Agent->>Agent: Compile findings into report
    Agent->>MCP: firm_export_github_pr<br/>(title="Security Audit Report", ...)
    MCP-->>Agent: {pr_url: "https://..."}
```

---

## 6. A2A Protocol Flow

The A2A (Agent-to-Agent) bridge enables inter-agent communication following
the A2A Protocol RC v1.0:

```mermaid
sequenceDiagram
    participant AgentA as Agent A (Client)
    participant Bridge as A2A Bridge<br/>(mcp-openclaw-ext)
    participant AgentB as Agent B (Remote)

    Note over AgentA,AgentB: 1. Discovery

    AgentA->>Bridge: openclaw_a2a_discovery<br/>(base_url="https://agent-b.example.com")
    Bridge->>AgentB: GET /.well-known/agent.json
    AgentB-->>Bridge: Agent Card (capabilities, skills, auth)
    Bridge-->>AgentA: {card: {...}, valid: true}

    Note over AgentA,AgentB: 2. Task Execution

    AgentA->>Bridge: openclaw_a2a_task_send<br/>(agent_url="...", message="Analyze this contract")
    Bridge->>Bridge: Validate input (SSRF check)
    Bridge->>AgentB: POST /a2a/tasks<br/>{"message": {"parts": [{"type":"text","text":"..."}]}}
    AgentB-->>Bridge: {"id": "task-123", "status": "working"}
    Bridge-->>AgentA: {task_id: "task-123", status: "working"}

    Note over AgentA,AgentB: 3. Status Polling

    AgentA->>Bridge: openclaw_a2a_task_status(task_id="task-123")
    Bridge->>AgentB: GET /a2a/tasks/task-123
    AgentB-->>Bridge: {"status": "completed", "artifacts": [...]}
    Bridge-->>AgentA: {status: "completed", artifacts: [...]}

    Note over AgentA,AgentB: 4. Cancellation (optional)

    AgentA->>Bridge: openclaw_a2a_task_cancel(task_id="task-456")
    Bridge->>AgentB: POST /a2a/tasks/task-456/cancel
    AgentB-->>Bridge: {"status": "canceled"}
    Bridge-->>AgentA: {ok: true}
```

---

## 7. CI/CD — PR Review Flow

The AI-powered PR review in `openclaw-review.yml`:

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant CI as GitHub Actions
    participant MCP as mcp-openclaw-extensions
    participant GW as OpenClaw Gateway

    Dev->>GH: Open PR to main

    GH->>CI: Trigger openclaw-review.yml

    CI->>CI: Collect PR diff (files changed)
    CI->>CI: Start mcp-openclaw-extensions<br/>(background process)
    CI->>CI: Wait for health check

    CI->>MCP: POST /mcp<br/>tools/call: "quality_review"
    MCP->>MCP: Analyze diff for:<br/>- Security issues<br/>- Code quality<br/>- Test coverage gaps<br/>- Documentation needs
    MCP-->>CI: Review findings

    alt Gateway available
        CI->>GW: Forward review to Gateway
        GW-->>CI: Enhanced review
    end

    CI->>GH: Create/update PR comment<br/>with AI review findings
    GH-->>Dev: Review notification
```

---

## 8. Cross-Project Memory Linking

Memory-os-ai supports linking memories across multiple projects:

```mermaid
graph TB
    subgraph Project_A["Project A (Fintech App)"]
        MA["Memory Index A<br/>API specs, architecture docs"]
    end

    subgraph Project_B["Project B (Mobile Client)"]
        MB["Memory Index B<br/>UI components, user stories"]
    end

    subgraph Project_C["Project C (Data Pipeline)"]
        MC["Memory Index C<br/>Schema definitions, ETL docs"]
    end

    subgraph MemoryEngine["Memory-os-ai Engine"]
        Links["Project Links<br/>memory_project_link()"]
        Search["Cross-project search<br/>memory_search()"]
    end

    MA <-->|linked| Links
    MB <-->|linked| Links
    MC <-->|linked| Links
    Links --> Search

    Search -->|"Query spans<br/>all linked projects"| Results["Combined Results<br/>(ranked by relevance)"]
```

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Mem as Memory-os-ai

    Dev->>Mem: memory_project_link<br/>(project_path="/projects/mobile-client")
    Mem->>Mem: Index linked project documents
    Mem-->>Dev: {linked: true, documents: 23}

    Dev->>Mem: memory_search<br/>(query="authentication flow")
    Mem->>Mem: Search across ALL linked projects
    Mem-->>Dev: Results from Project A (API spec)<br/>+ Project B (login screen)<br/>+ Project C (user table schema)
```

---

## 9. Hebbian Memory — Adaptive Learning

The Hebbian memory module in `mcp-openclaw-extensions` implements adaptive
weight adjustment for tool co-activation patterns:

```mermaid
flowchart TB
    subgraph Harvest["1. Harvest"]
        Logs["JSONL tool logs"]
        Parse["Parse tool call events"]
        CoAct["Identify co-activations<br/>(tools called within<br/>same session window)"]
    end

    subgraph Learn["2. Weight Update"]
        Matrix["Co-activation matrix"]
        Hebb["Hebbian rule:<br/>Δw = η · a_i · a_j"]
        Decay["Exponential decay<br/>on unused connections"]
    end

    subgraph Analyze["3. Analysis"]
        Clusters["Tool clusters<br/>(frequently co-used)"]
        Drift["Drift detection<br/>(pattern changes over time)"]
        PII["PII stripping<br/>(privacy protection)"]
    end

    Logs --> Parse --> CoAct
    CoAct --> Matrix --> Hebb --> Decay
    Decay --> Clusters
    Decay --> Drift
    Decay --> PII
```
