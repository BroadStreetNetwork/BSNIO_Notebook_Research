# BSNIO Notebook Research — Implementation Guide (V2.1)

This guide describes the Bun/TypeScript implementation approach, emphasizing adapters and deployment profiles.

## 0) Setup checklist (fast)
1) Pick a deployment profile
2) Pick relational DB + object storage + vector/RAG backend
3) Configure auth
4) Configure LLM and artifact providers
5) Confirm `/api/v1/capabilities` shows ready components

## 1) Suggested repo layout
- apps/web — Next.js UI
- apps/api — Bun API (REST + SSE/WebSocket)
- apps/worker — background jobs
- packages/core — types, capability schema, domain models
- packages/adapters — provider/data/auth adapters
- deploy — deployment profiles and scripts
- docs — documentation

## 2) Deployment profiles
- **abacus-hosted**: API + worker in Abacus environment; optional Abacus DB/vector/storage; optional RouteLLM routing
- **vps-docker**: Docker Compose for web/api/worker; optional postgres/mysql/qdrant services
- **netlify-web**: web on Netlify; API/worker on VPS or Abacus; configure CORS + OAuth callbacks

## 3) Data plane configuration (3 independent choices)

### 3.1 Relational DB (metadata)
Pick one:
- Postgres / Supabase Postgres
- MySQL
- SQLite
- Turso
- DuckDB
- Abacus transactional DB (optional)

### 3.2 Object storage (files/artifacts)
Pick one:
- Supabase Storage
- S3-compatible (S3/R2/etc.)
- Abacus Storage (optional)
- Local filesystem (VPS/dev)

### 3.3 Vector store / RAG index (retrieval)
Pick one:
- Gemini File Search (Gemini File Store)
- pgvector (requires Postgres)
- sqlite-vector (requires SQLite/Turso)
- Qdrant
- LanceDB
- Abacus Vector Store (optional)
- vectorwrap (optional)

### 3.4 Recommended combos
- MySQL + Gemini File Store + S3/R2
- SQLite/Turso + Gemini File Store + S3/R2
- SQLite/Turso + sqlite-vector + local_fs (VPS simplicity)
- Postgres + pgvector + S3/R2
- Any relational + Qdrant (dedicated vector DB)
- Any relational + LanceDB (embedded/local-first)

## 4) Providers (LLM + artifacts)

### 4.1 LLM adapters
Implement:
- `OpenAICompatibleLLMAdapter` (OpenAI/OpenRouter/RouteLLM via base URL)
- `GeminiLLMAdapter`
- `AnthropicLLMAdapter`

### 4.2 Artifact adapters
Implement interfaces:
- `TTSProvider`
- `ImageProvider`
- `VideoProvider` (optional)

## 5) Auth
Choose one:
- Supabase Auth
- OIDC (Google/GitHub)
- JWT-only (private SSO)
- Dev/no-auth

## 6) Capabilities endpoint
Implement `GET /api/v1/capabilities`.

Checks:
- relational connectivity
- storage write test
- vector readiness + constraint evaluation
- at least one LLM provider configured

UI must be capability-driven.

## 7) Retrieval implementations

### 7.1 Gemini File Store
- store per notebook
- upload docs with metadata
- query with metadata filters; return citations

### 7.2 pgvector
- vector column + index
- similarity search returns chunk IDs; join to source metadata

### 7.3 sqlite-vector
- sqlite-vector extension
- similarity search over chunk table

### 7.4 Qdrant / LanceDB / Abacus
- collection per notebook
- upsert chunk vectors + metadata payload
- query top-k; join back to relational

### 7.5 vectorwrap
- optional wrapper to unify multiple vector stores behind one API

## 8) MCP ingestion
- connector registry
- per-notebook enabling and permissions
- provenance in source metadata
- audit log

## 9) Observability
Log per request:
- provider/model
- tokens/cost/latency
- retrieval stats
- artifact generation time/size
- errors/retries
