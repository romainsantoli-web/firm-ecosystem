# ADR-004: Pydantic v2 for All Tool Inputs

## Status

Accepted — 2026-01-10

## Context

138 MCP tools accept JSON arguments from untrusted clients. Every input must be
validated before processing.

Options considered:
1. **Pydantic v2** — Python data validation with type hints
2. **JSON Schema validation** (jsonschema library)
3. **Manual validation** (if/else checks in each handler)
4. **marshmallow** — serialization/deserialization library

## Decision

Use **Pydantic v2** with a `TOOL_MODELS` registry mapping tool names to model classes.

## Rationale

- **Type safety**: Python type hints as the single source of truth
- **Path traversal guard**: `ConfigPathInput` base class blocks `..` in all file paths
- **Cross-field validation**: `@model_validator(mode="after")` for complex constraints
- **Performance**: Pydantic v2 (Rust core) is 5-50x faster than v1
- **Consistent with Memory-os-ai**: Both servers use the same validation pattern
- **Error messages**: Pydantic produces clear, structured validation errors
- **138 models maintained in one file** (`models.py`) for centralized review

## Consequences

- **Positive**: Every tool input is validated before handler execution
- **Positive**: Path traversal attacks blocked at the model level
- **Negative**: `models.py` is 1,689 lines — large but centralized
- **Related**: Coverage tests verify TOOL_MODELS matches TOOL_REGISTRY
