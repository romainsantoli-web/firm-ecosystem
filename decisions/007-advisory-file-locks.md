# ADR-007: Advisory File Locks for Workspace Safety

## Status

Accepted — 2026-02-28

## Context

Multiple agents or MCP clients may try to modify the same workspace concurrently.
Need a locking mechanism that:
- Survives process crashes (auto-release)
- Works on macOS and Linux
- Has no external dependencies

Options considered:
1. **`fcntl.LOCK_EX | LOCK_NB`** — POSIX advisory file locks
2. **In-memory mutex** (threading.Lock)
3. **Redis distributed lock** (Redlock)
4. **File-based PID lock** (write PID to lockfile)

## Decision

Use **POSIX advisory file locks** via `fcntl.LOCK_EX | LOCK_NB`.

## Rationale

- **Crash-safe**: OS automatically releases locks when the process dies
- **No dependencies**: Built into Python's `fcntl` module
- **Non-blocking option**: `LOCK_NB` allows immediate failure instead of hanging
- **Cross-platform**: Works on macOS and Linux (not Windows, but not a target)
- **Owner tracking**: Lock metadata (owner string, timestamp) written to the lockfile

## Consequences

- **Positive**: Prevents concurrent workspace corruption
- **Positive**: Auto-cleanup on crash
- **Negative**: Advisory only — processes that don't check the lock can still write
- **Negative**: Not available on Windows
- **Related**: ADR-001 — JSON file persistence uses same atomic write pattern
