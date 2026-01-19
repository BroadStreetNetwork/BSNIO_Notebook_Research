# BSNIO Notebook Research — Docs Update Plan (2026, V2.1)

This plan updates V1 assumptions into a flexible, multi-provider, deploy-anywhere architecture.

## Goals
- Bun/TypeScript refactor
- Provider abstraction (LLM/TTS/Image/Video)
- Data plane split (Relational / Storage / Vector-RAG)
- Auth abstraction (Supabase, OIDC, JWT)
- Deployment profiles (Abacus, VPS, Netlify, optional Vercel)
- Capabilities introspection

## Updated option sets (must match across all docs)

### Relational DB
- Postgres / Supabase Postgres
- MySQL
- SQLite
- Turso
- DuckDB
- Abacus transactional DB (optional)

### Vector/RAG
- Gemini File Search (Gemini File Store)
- pgvector (Postgres only)
- sqlite-vector (SQLite/Turso only)
- Qdrant
- LanceDB
- Abacus Vector Store
- vectorwrap (optional)

### Storage
- Supabase Storage
- S3-compatible (S3/R2/etc.)
- Abacus Storage
- Local FS

### Compatibility constraints
- pgvector requires Postgres
- sqlite-vector requires SQLite/Turso

## Workstreams

### WS1 — Capabilities + config validation
- canonical schema (Doc 05)
- `/api/v1/capabilities`
- readiness checks + constraint evaluation

### WS2 — Provider layer
- OpenAI-compatible adapter (OpenAI/OpenRouter/RouteLLM)
- Gemini adapter
- Anthropic adapter
- image generation as first-class capability
- expand TTS/video provider interfaces

### WS3 — Data plane adapters
- relational adapters: postgres/supabase/mysql/sqlite/turso/duckdb/abacus
- storage adapters: supabase/s3/abacus/local
- vector adapters: gemini/pgvector/sqlite-vector/qdrant/lancedb/abacus/vectorwrap
- indexing pipeline + export/import behavior across backends

### WS4 — Auth + sharing
- auth adapters + stable permission layer
- share notebooks/sources/chats/artifacts

### WS5 — Outputs
- codebase understanding outputs (repo wiki)
- docs/how-to/SOPs
- prompt packs
- study artifacts + multimodel chat

### WS6 — Deployment recipes
- Abacus-hosted recipe
- VPS Docker recipe
- Netlify web + API recipe
- validate using capabilities endpoint
