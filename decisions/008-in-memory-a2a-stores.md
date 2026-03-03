# ADR-008: In-Memory Stores for A2A Protocol

## Status

Accepted — 2026-03-05

## Context

The A2A bridge needs to store tasks, push notification configs, and subscription
state. The MCP server is single-process.

Options considered:
1. **In-memory dicts** (`_TASKS`, `_PUSH_CONFIGS`)
2. **SQLite** database
3. **Redis** cache
4. **JSON file** persistence (like ACP bridge)

## Decision

Use **in-memory Python dicts** for A2A protocol stores.

## Rationale

- **Single-process**: The MCP server is a single aiohttp process — no need for
  cross-process storage
- **Ephemeral by design**: A2A tasks are short-lived (seconds to minutes)
- **Zero I/O**: No disk writes for transient protocol state
- **Simplicity**: Dict lookup is O(1), no serialization needed
- **Push configs** indexed by `(task_id, push_id)` tuple

## Consequences

- **Positive**: Fastest possible access, no serialization overhead
- **Negative**: Data lost on server restart
- **Mitigation**: A2A tasks can be re-sent by the client after restart
- **Future**: If persistence is needed, migrate to SQLite (similar to Hebbian memory)
