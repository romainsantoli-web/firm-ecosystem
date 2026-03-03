# ADR-003: aiohttp over FastAPI for MCP Server

## Status

Accepted — 2026-01-10

## Context

The mcp-openclaw-extensions server needs an HTTP framework for the JSON-RPC 2.0
endpoint and SSE streaming.

Options considered:
1. **aiohttp** — low-level async HTTP server
2. **FastAPI** — high-level framework with OpenAPI, validation, dependency injection
3. **Starlette** — ASGI framework (FastAPI's underlying layer)

## Decision

Use **aiohttp** for the MCP server.

## Rationale

- **Full control over JSON-RPC dispatch**: MCP uses a single endpoint (`POST /mcp`)
  with method routing — FastAPI's path-based routing adds unnecessary abstraction
- **Native SSE support**: `aiohttp.web.StreamResponse` is ideal for SSE without
  third-party libraries
- **Minimal overhead**: No OpenAPI schema generation, no dependency injection,
  no middleware stack — the server does one thing
- **Pydantic handled separately**: Tool validation is done via `TOOL_MODELS` dict,
  not via FastAPI's built-in validation
- **Lightweight dependency**: aiohttp has fewer transitive dependencies than FastAPI

## Consequences

- **Positive**: Full control, minimal dependencies, simple deployment
- **Negative**: Manual JSON-RPC parsing (no framework support)
- **Negative**: No auto-generated API documentation (mitigated by this docs repo)
