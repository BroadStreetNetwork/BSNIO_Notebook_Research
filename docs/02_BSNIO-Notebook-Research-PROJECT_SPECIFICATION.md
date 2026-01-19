# BSNIO Notebook Research — Project Specification (V2.1)

## 1) Scope
A NotebookLM-style application that:
- ingests heterogeneous sources via uploads + URLs + MCP connectors
- stores notebook metadata in a configurable **relational DB**
- stores files and artifacts in configurable **object storage**
- performs grounded retrieval through a configurable **vector/RAG backend**
- supports multiple LLM providers and multi-model chats
- produces reusable outputs: grounded answers with citations, wikis/docs/how-tos, prompt packs, tables, slide outlines, flashcards/quizzes, audio, images, optional video

## 2) System architecture

### 2.1 Runtime (default)
- **Web**: Next.js
- **API**: Bun (TypeScript) REST + streaming (SSE/WebSocket)
- **Worker**: background jobs (ingestion/indexing/artifact generation)

### 2.2 Core abstractions

#### A) Relational store (metadata + transactions)
Purpose: users, notebooks, source metadata, chats, artifacts, permissions, jobs, usage logs.

Supported adapters:
- Postgres (self-hosted)
- Supabase Postgres
- MySQL
- SQLite
- Turso (hosted SQLite)
- DuckDB (embedded analytics)
- Abacus transactional DB (optional adapter)

#### B) Object storage (files)
Purpose: raw uploads, extracted text, transcripts, generated artifacts.

Supported adapters:
- Supabase Storage
- S3-compatible (S3/R2/etc.)
- Abacus Storage (optional adapter)
- Local filesystem (VPS/dev)

#### C) Vector store / RAG index (retrieval)
Purpose: semantic retrieval + metadata filters, optional hybrid retrieval.

Supported adapters:
- Gemini File Search ("Gemini File Store")
- pgvector (Postgres only)
- sqlite-vector (SQLite/Turso only)
- Qdrant
- LanceDB
- Abacus Vector Store (optional adapter)
- vectorwrap (optional wrapper/unifier)

**Independence rule:** Relational DB and Vector/RAG provider are configured separately.

**Constraints (enforced by config validation + capabilities):**
- `pgvector` requires Postgres/Supabase Postgres
- `sqlite-vector` requires SQLite/Turso

#### D) Provider layer (LLM + media)
LLM providers:
- Gemini, Anthropic, OpenAI, OpenRouter, Abacus RouteLLM

Media providers:
- TTS providers (pluggable)
- Image providers (pluggable)
- Video providers (pluggable)

#### E) Auth layer
- Supabase Auth
- OAuth/OIDC (Google/GitHub)
- JWT-only for private deployments behind SSO
- Dev/no-auth profile

#### F) Deployment profiles
- Abacus-hosted
- VPS Docker
- Netlify web + external API/worker
- Vercel (optional)

## 3) Capabilities system

### 3.1 Endpoint
`GET /api/v1/capabilities`

### 3.2 Responsibilities
- UI hides/disables features not configured
- API returns actionable configuration errors
- Admin setup checklist derived from this object
- Enforces relational/vector compatibility constraints

### 3.3 Capability domains
- `data.relational` (provider, ready, details)
- `data.storage` (provider, ready, details)
- `data.vector` (provider, ready, constraints[], details)
- `providers.llm` (enabled[], default, model_registry)
- `providers.tts`, `providers.image`, `providers.video`
- `auth` (mode, providers)
- `features` (sharing, artifacts, mcp, multimodel_chat, codebase_tools)
- `deploy` (profile)

## 4) Workflows

### 4.1 Notebook creation
- Create notebook in relational DB
- Initialize vector index state:
  - Gemini: create store id, persist pointer
  - Qdrant/LanceDB/Abacus/vectorwrap: create collection/table, persist pointer
  - pgvector/sqlite-vector: ensure vector tables and indexes exist
- Assign ownership + default permissions

### 4.2 Source ingestion
Channels:
- file upload (pdf/docx/md/txt/csv)
- URL import (crawler)
- MCP connectors (Drive, GitHub repos, Notion, etc.)

Pipeline:
1) store raw file in object storage
2) extract text + metadata (optionally chunk)
3) index into vector provider OR upload to Gemini store
4) persist source record + ingestion status + provenance
5) record indexing metrics

### 4.3 Retrieval + chat (grounded)
- question → retrieval plan
- retrieve evidence using notebook’s configured vector provider
- generate response using selected LLM provider/model
- return citations/grounding (source IDs + offsets + URLs)
- persist message + usage/cost + retrieval stats

### 4.4 Multi-model chat
- per-message model selection OR routing (RouteLLM)
- store model used, tokens, cost, latency per message
- optional draft+verify mode

### 4.5 Artifact generation
Artifact types:
- answers w/ citations (saved)
- wiki/doc sets
- how-to/SOPs
- prompt/instruction packs
- tables (structured)
- slide outlines
- flashcards/quizzes
- audio (TTS)
- images
- optional video

Lifecycle:
- queued → running → completed/failed
- stored in object storage; linked to notebook + source set + prompt template id
- shareable via share links/permissions

## 5) Data model (logical)

### 5.1 Core tables
- users
- workspaces (optional)
- notebooks
- sources
- chat_sessions
- chat_messages
- artifacts
- artifact_files
- index_jobs
- usage_logs
- share_links
- permissions
- provider_keys (encrypted at rest)

### 5.2 Vector pointers (stored in relational DB)
- notebooks.vector_provider
- notebooks.vector_ref (Gemini store id / Qdrant collection / Abacus ref / etc.)

### 5.3 Optional chunk table (app-managed chunking)
Used when vector provider requires it (pgvector/sqlite-vector/qdrant/lancedb/abacus/vectorwrap):
- source_chunks (source_id, text, metadata, embedding/vector reference)

Gemini File Store can skip chunk persistence, but storing extracted text is recommended for transparency/export.

## 6) Configuration

### 6.1 Data plane
RELATIONAL_PROVIDER = postgres | supabase_postgres | mysql | sqlite | turso | duckdb | abacus_relational
RELATIONAL_DSN = DSN or file path

STORAGE_PROVIDER = supabase_storage | s3 | abacus_storage | local_fs

VECTOR_PROVIDER = gemini_file_search | pgvector | sqlite_vector | qdrant | lancedb | abacus_vector | vectorwrap

### 6.2 Auth
AUTH_PROVIDER = supabase_auth | oidc | jwt | dev

### 6.3 LLM + media keys
- GEMINI_API_KEY
- ANTHROPIC_API_KEY
- OPENAI_API_KEY
- OPENROUTER_API_KEY (+ OPENROUTER_BASE_URL)
- ROUTELLM_API_KEY (+ ROUTELLM_BASE_URL)

## 7) Non-functional requirements
- Security: encrypt provider keys, least-privilege service accounts
- Reliability: retries/circuit breakers, idempotent jobs
- Performance: streaming, caching, chunking strategies per backend
- Portability: runnable on VPS; minimal mandatory managed dependencies
