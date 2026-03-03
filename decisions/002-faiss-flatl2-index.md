# ADR-002: FAISS FlatL2 for Vector Index

## Status

Accepted — 2026-01-15

## Context

Memory-os-ai needs a vector similarity search index for semantic document search.
The index must work on consumer hardware (4-8 GB RAM, no GPU required).

Options considered:
1. **FAISS FlatL2** — exact nearest-neighbor with L2 distance
2. **FAISS HNSW** — approximate nearest-neighbor with graph-based index
3. **ChromaDB** — vector database with built-in persistence
4. **Qdrant** — client-server vector database

## Decision

Use **FAISS FlatL2** as the default index type.

## Rationale

- **Exact results**: No approximation or parameter tuning needed
- **Zero configuration**: Works out of the box with any dimensionality
- **Portable**: Single `.index` file, easy to backup and transfer
- **Low memory**: 384d vectors × 100K docs ≈ 150 MB RAM
- **No server**: Runs in-process, no additional service to manage
- **Upgradable**: Can switch to HNSW for > 1M vectors by changing one parameter

## Consequences

- **Positive**: Simple, predictable, no tuning required
- **Negative**: Linear scan O(n) — becomes slow at > 1M vectors
- **Mitigation**: `memory_compact` tool reduces index size by deduplication
