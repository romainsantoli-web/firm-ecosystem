# 009 — SQLite Lazy Proxy Initialization

- **Status**: Accepted
- **Date**: 2026-03-03
- **Context**: TaskFlow Pro SaaS MVP
- **Decision**: Use a JavaScript `Proxy` for lazy SQLite database initialization
- **Drivers**: Test isolation, environment variable timing

## Context

better-sqlite3 opens the database file at constructor time. When modules are imported
during test setup, the database is created before test environment variables (`DB_PATH`,
`JWT_SECRET`) are set — causing tests to use the wrong database path.

## Decision

Wrap the database singleton in a `Proxy` object that delays actual initialization until
the first property access:

```typescript
const db: DatabaseType = new Proxy({} as DatabaseType, {
  get(_target, prop) {
    const instance = getDb(); // first call creates the DB
    const val = (instance as unknown as Record<string | symbol, unknown>)[prop];
    if (typeof val === 'function') return val.bind(instance);
    return val;
  },
});
```

## Consequences

- **Positive**: Tests can set `DB_PATH=/tmp/test.db` before any DB operation
- **Positive**: Zero overhead after first access (same instance reused)
- **Positive**: No API change — callers use `db.prepare()` as usual
- **Negative**: TypeScript requires explicit type annotation to satisfy `declaration: true`
- **Alternative rejected**: Dependency injection (too much refactoring for an MVP)
