# ADR-001: JSON File Persistence for ACP Sessions

## Status

Accepted — 2026-02-28

## Context

The ACP (Agent Communication Protocol) bridge needs to persist session mappings
(`run_id → gateway_session_key`) to survive process crashes and restarts.

Options considered:
1. **JSON file** with `os.replace()` atomic writes
2. **SQLite** database
3. **Redis** in-memory store

## Decision

Use **JSON file persistence** with atomic writes via `os.replace()`.

## Rationale

- **Zero dependencies**: No Redis or SQLite library required
- **Sufficient for scale**: Handles ≤ 10K sessions without performance issues
- **Atomic writes**: `os.replace()` is POSIX-atomic, prevents corruption on crash
- **Human-readable**: JSON files can be inspected and debugged manually
- **Reversible**: Easy to migrate to SQLite or Redis later if needed

## Consequences

- **Positive**: Simpler deployment, no infrastructure dependencies
- **Negative**: Not suitable for > 10K concurrent sessions (JSON parsing becomes slow)
- **Negative**: No built-in expiration (handled by application-level TTL checks)

## Related

- `acp_bridge.py` — implementation
- ADR-007 — advisory locks for concurrent access
