# ADR-005: Split Hebbian Memory into Submodules

## Status

Accepted — 2026-03-07

## Context

The `hebbian_memory.py` module grew to 1,560 lines with 8 tool handlers covering
harvest, weight updates, analysis, validation, PII checks, and drift detection.

## Decision

Split into 5 submodules under `src/hebbian_memory/`:

```
hebbian_memory/
├── __init__.py      # Public API: TOOLS list, handler imports
├── _helpers.py      # Shared utilities (DB connections, constants)
├── _runtime.py      # harvest, weight_update, status
├── _analysis.py     # analyze, drift_check
└── _validation.py   # layer_validate, pii_check, decay_config_check
```

## Rationale

- **Maintainability**: ~300 lines per file instead of 1,560
- **Backward compatibility**: `from src.hebbian_memory import TOOLS` still works
- **Logical grouping**: Runtime (hot path), Analysis (cold path), Validation (checks)
- **Testability**: Each submodule can be tested independently

## Consequences

- **Positive**: Easier to review, maintain, and extend
- **Positive**: Better git blame — changes isolated to relevant submodule
- **Negative**: One more package in the import chain (negligible)
