# BSNIO Notebook Research
## Project Specification (V2)

> **Version:** 2.0 (Docs Refresh) | **Date:** January 2026
>
> This specification defines the **portable architecture**, **core data model**, **API surface**, and **abstraction layers** for a NotebookLM-style research application that supports:
> - Multiple LLM providers (Gemini, Claude, OpenAI, OpenRouter, Abacus RouteLLM)
> - Multiple retrieval backends (Gemini File Search, Postgres+pgvector, Abacus Vector Store)
> - Multiple storage/auth/deployment profiles
> - Shareable notebooks, sources, chats, and generated artifacts
> - Capability-gated features (introspection)

**Related docs:**
- `01_BSNIO-Notebook-Research_VISION_DOCUMENT.md`
- `03_BSNIO-Notebook-Research-IMPLEMENTATION_GUIDE.md`
- `04_BSNIO-Notebook-Research_DOCS_UPDATE_PLAN_2026_v2.md`
- `05_BSNIO-Notebook-Research_CAPABILITIES_PROVIDERS_DEPLOYMENT.md`

---

## 1) Scope and Non-Goals

### 1.1 In scope
- Source-centric notebooks (files, URLs, YouTube, pasted text, repo/codebase sources)
- RAG chat with citations (and a safe “no-RAG” fallback)
- Studio outputs/artifacts (reports, slides, flashcards, quizzes, mind maps, audio/video, wiki/docs/how-to, prompt packs)
- Sharing: notebook share links + artifact share links + shared chat sessions (viewer/editor, link/invite)
- Multi-provider and multi-model chat: per-message or per-session model selection
- Usage/cost/storage metrics and audit logging
- Portable deployments via “Deployment Profiles”

### 1.2 Explicit non-goals (for V2 docs)
- Full “local desktop app” packaging is optional future work; V2 docs focus on web-first deployments.
- Re-creating Google’s proprietary internal tooling is out of scope; we provide open analogs.

---

## 2) High-Level Architecture

### 2.1 System components
- **Frontend** (web UI): Next.js (recommended), or any client that can talk to the API
- **Backend API** (Bun + TypeScript): REST + optional streaming
- **Job Runner** (optional): background tasks for ingestion, chunking, embeddings, artifact generation
- **Data Layer**: DB + Storage adapters
- **Provider Layer**: LLM/TTS/Image/Video/Search adapters
- **Retrieval Layer**: Gemini File Search, pgvector, Abacus Vector Store (adapter-based)

### 2.2 Abstraction layers (required)
1) **Provider Abstraction Layer**
   - Chat/completions, embeddings, rerank, image generation, TTS, video, web search
2) **Data Abstraction Layer**
   - Transactional DB adapter
   - Object storage adapter
   - Optional cache
3) **Auth Abstraction Layer**
   - Identity provider adapters (Supabase Auth, OAuth/OIDC, JWT-only)
   - Authorization policies (RBAC + sharing permissions)
4) **Deployment Abstraction Layer** (aka Deployment Profiles)
   - A small set of profiles mapping env vars → enabled capabilities

### 2.3 Capability system
The backend computes a canonical `Capabilities` object at startup and exposes it via API.

See `05_BSNIO-Notebook-Research_CAPABILITIES_PROVIDERS_DEPLOYMENT.md` for schema and examples.

---

## 3) Core Concepts and Data Model

### 3.1 Entities
- **User**: authenticated identity
- **Notebook**: container for sources + chats + artifacts
- **Source**: a piece of input (PDF, doc, URL, transcript, repo file set)
- **Chat Session**: conversation scoped to a notebook (or global)
- **Chat Message**: user/assistant turns with citations + model usage
- **Artifact**: generated outputs (reports/slides/flashcards/mindmaps/audio/video/wiki/prompt packs)
- **Share**: public link or invite-based sharing for notebooks/chats/artifacts
- **Usage Log**: token/cost metrics and operational audit

### 3.2 Retrieval modes
A notebook selects one retrieval mode (can be changed, but may require re-indexing):
- `none` (no-RAG)
- `gemini_file_search`
- `pgvector`
- `abacus_vector_store`

---

## 4) Database Schema (Canonical)

> This schema is **provider-neutral**. Implementations can use Supabase Postgres (with RLS) or direct Postgres (app-layer auth). For `pgvector` mode, enable the `vector` extension.

### 4.1 Core tables

#### 4.1.1 `profiles`
- `id` (uuid, pk)
- `email` (text)
- `name` (text)
- `avatar_url` (text)
- `settings` (jsonb)
- `created_at`, `updated_at`

#### 4.1.2 `notebooks`
- `id` (uuid, pk)
- `owner_user_id` (uuid, fk → profiles.id)
- `name` (text)
- `description` (text)
- `emoji` (text)
- `settings` (jsonb)
  - includes: persona defaults, default model, retrieval config, ui layout preferences
- `retrieval_mode` (text) one of: `none|gemini_file_search|pgvector|abacus_vector_store`
- `external_retrieval_store_id` (text, nullable) (e.g. Gemini store id / Abacus retriever id)
- `created_at`, `updated_at`

#### 4.1.3 `sources`
- `id` (uuid, pk)
- `notebook_id` (uuid, fk)
- `type` (text) e.g. `pdf|docx|txt|url|youtube|text|repo|audio|image`
- `name` (text)
- `status` (text) `pending|processing|ready|failed`
- `storage_object_key` (text, nullable) pointer to object storage
- `mime_type` (text)
- `bytes` (bigint)
- `token_count` (int, nullable)
- `content_hash` (text, nullable) for dedupe
- `metadata` (jsonb) (url, transcript, repo info, etc.)
- `source_guide` (jsonb) (summary/topics/suggested questions)
- `external_doc_id` (text, nullable) for retrieval backend
- `created_at`, `updated_at`

#### 4.1.4 `chat_sessions`
- `id` (uuid, pk)
- `notebook_id` (uuid, fk, nullable for global chat)
- `created_by_user_id` (uuid)
- `title` (text)
- `settings` (jsonb)
  - per-session defaults: model/provider, system prompt/persona, retrieval overrides
- `created_at`, `updated_at`

#### 4.1.5 `chat_messages`
- `id` (uuid, pk)
- `session_id` (uuid, fk)
- `role` (text) `user|assistant|system|tool`
- `content` (text)
- `citations` (jsonb) normalized citations
- `source_ids_used` (jsonb)
- `model_used` (text)
- `provider_used` (text)
- `input_tokens` (int)
- `output_tokens` (int)
- `cost_usd` (numeric)
- `latency_ms` (int)
- `created_at`

#### 4.1.6 `artifacts`
> Replaces “audio_overviews/video_overviews/etc” with a unified artifact model.

- `id` (uuid, pk)
- `notebook_id` (uuid, fk)
- `created_by_user_id` (uuid)
- `type` (text) e.g. `report|slides|flashcards|quiz|mind_map|audio|video|wiki|howto|prompt_pack|codebase_doc`
- `title` (text)
- `status` (text) `pending|running|completed|failed|canceled`
- `progress` (jsonb)
- `source_ids` (jsonb)
- `generation_params` (jsonb)
- `result_text` (text, nullable)
- `result_json` (jsonb, nullable)
- `storage_object_key` (text, nullable)
- `thumbnail_object_key` (text, nullable)
- `model_used` (text)
- `provider_used` (text)
- `input_tokens` (int)
- `output_tokens` (int)
- `cost_usd` (numeric)
- `created_at`, `completed_at`

#### 4.1.7 `shares`
> A single table for share links + invite shares.

- `id` (uuid, pk)
- `type` (text) `notebook|chat_session|artifact|source`
- `resource_id` (uuid)
- `owner_user_id` (uuid)
- `access` (text) `restricted|invite|link`
- `permission` (text) `viewer|commenter|editor`
- `share_token` (text, unique, nullable) for link sharing
- `expires_at` (timestamptz, nullable)
- `requires_auth` (bool)
- `settings` (jsonb)
  - allow_chat_in_shared_notebook, allow_download_sources, allow_export, watermark
- `created_at`, `revoked_at`

#### 4.1.8 `share_invites`
- `id` (uuid, pk)
- `share_id` (uuid, fk)
- `invitee_email` (text)
- `invitee_user_id` (uuid, nullable)
- `status` (text) `pending|accepted|revoked`
- `created_at`, `accepted_at`

#### 4.1.9 `usage_logs`
- `id` (uuid, pk)
- `user_id` (uuid)
- `notebook_id` (uuid, nullable)
- `operation_type` (text) e.g. `chat|ingest|embed|artifact.generate|artifact.download`
- `provider` (text)
- `model` (text)
- `input_tokens`, `output_tokens` (int)
- `cost_usd` (numeric)
- `storage_bytes_in`, `storage_bytes_out` (bigint)
- `metadata` (jsonb)
- `created_at`

#### 4.1.10 `user_provider_keys` (optional)
> Enables user-level BYO keys. Values must be encrypted at rest (envelope encryption).

- `id` (uuid, pk)
- `user_id` (uuid)
- `provider` (text)
- `key_label` (text)
- `key_ciphertext` (text)
- `key_kms_key_id` (text, nullable)
- `created_at`, `revoked_at`

### 4.2 pgvector tables (only for `retrieval_mode = pgvector`)

#### 4.2.1 `documents`
- `id` (uuid, pk)
- `notebook_id` (uuid)
- `source_id` (uuid)
- `title` (text)
- `metadata` (jsonb)
- `created_at`

#### 4.2.2 `chunks`
- `id` (uuid, pk)
- `document_id` (uuid)
- `chunk_index` (int)
- `content` (text)
- `content_tsv` (tsvector) (keyword search)
- `embedding` (vector(N))
- `token_count` (int)
- `metadata` (jsonb)

Indexes:
- `ivfflat` or `hnsw` on `embedding` (depending on extension/support)
- GIN index on `content_tsv`

---

## 5) Retrieval / RAG Specification

### 5.1 Retrieval provider interface
All retrieval backends implement:
- `ingestSource(source) -> { external_doc_id }`
- `delete(external_doc_id)`
- `query({ notebook_id, query, filters, top_k }) -> { chunks[], citations[] }`
- `health()`

### 5.2 Backend A: Gemini File Search
- Use Gemini’s file search store per notebook (or per org), store `external_retrieval_store_id`.
- Store **only metadata + pointers** in Postgres; retrieval happens via Gemini.
- Support **metadata filters** when the backend supports it.

### 5.3 Backend B: Postgres + pgvector
- App extracts text, chunks, embeds, stores in `chunks.embedding`
- Supports:
  - similarity search (pgvector)
  - keyword search (tsvector)
  - metadata filters (jsonb)
  - hybrid retrieval (blend keyword + vector)

### 5.4 Backend C: Abacus Vector Store
- Use Abacus Vector Store / document retriever as the chunk store.
- Store Abacus retriever id + external doc ids; perform retrieval via Abacus.

---

## 6) Provider Abstraction Layer

### 6.1 Provider categories
- LLM chat/completions
- Embeddings
- Rerank (optional)
- Image generation
- TTS
- Video generation
- Web search / browsing (optional)

### 6.2 Provider selection rules
- A request can specify provider/model at:
  - app default
  - notebook default
  - session default
  - message override
- Policy constraints:
  - deployment admin can restrict allowed providers/models
  - shared notebooks can restrict to “read-only inference” (no tool calls)

### 6.3 Cost accounting
- Do not hardcode costs in docs.
- Implementation must support:
  - provider-reported usage (preferred)
  - optional pricing registry (config file) to estimate costs

---

## 7) Artifacts / Studio Outputs

### 7.1 Artifact types (initial)
- Study: `flashcards`, `quiz`, `study_guide`, `faq`
- Explain: `report`, `briefing`, `timeline`, `compare`, `critique`
- Visual: `mind_map`, `slides`
- Audio/video: `audio`, `video`
- Build: `wiki`, `howto`, `prompt_pack`, `codebase_doc`

### 7.2 Artifact generation contract
Each artifact generation request includes:
- `type`
- `source_ids` (or “all sources”)
- optional `instructions`
- optional output format params (language, tone, length)
- provider/model override (optional)

Artifacts store:
- generation params
- usage/cost
- citations (for text outputs)
- pointers to files (for audio/video/slides)

---

## 8) Sharing and Permissions

### 8.1 Share modes
- Restricted (owner only)
- Invite-based sharing (viewer/editor)
- Public link sharing (tokenized)

### 8.2 Shared notebook behavior
Configurable per share:
- allow chat within shared notebook
- allow viewing sources list
- allow downloading sources
- allow exporting notebooks
- watermark outputs

---

## 9) API Surface (V2)

> The API is stable, versioned: `/api/v1/...`.

### 9.1 Capabilities
- `GET /api/v1/capabilities`
- `GET /api/v1/admin/capabilities`

### 9.2 Notebooks
- `POST /api/v1/notebooks`
- `GET /api/v1/notebooks`
- `GET /api/v1/notebooks/:id`
- `PATCH /api/v1/notebooks/:id`
- `DELETE /api/v1/notebooks/:id`

### 9.3 Sources
- `POST /api/v1/notebooks/:id/sources/upload`
- `POST /api/v1/notebooks/:id/sources/url`
- `POST /api/v1/notebooks/:id/sources/youtube`
- `POST /api/v1/notebooks/:id/sources/text`
- `POST /api/v1/notebooks/:id/sources/repo` (GitHub/MCP ingestion)
- `GET /api/v1/notebooks/:id/sources`
- `GET /api/v1/notebooks/:id/sources/:sid`
- `DELETE /api/v1/notebooks/:id/sources/:sid`

### 9.4 Chat
- `POST /api/v1/notebooks/:id/chat/sessions`
- `GET /api/v1/notebooks/:id/chat/sessions`
- `GET /api/v1/chat/sessions/:sid/messages`
- `POST /api/v1/chat/sessions/:sid/messages`
- `POST /api/v1/chat/sessions/:sid/stream` (optional)

### 9.5 Artifacts
- `POST /api/v1/notebooks/:id/artifacts`
- `GET /api/v1/notebooks/:id/artifacts`
- `GET /api/v1/artifacts/:aid`
- `GET /api/v1/artifacts/:aid/download` (if file-backed)
- `DELETE /api/v1/artifacts/:aid`

### 9.6 Sharing
- `POST /api/v1/shares` (create share)
- `GET /api/v1/shares/:token` (resolve)
- `POST /api/v1/shares/:token/accept` (invite)
- `POST /api/v1/shares/:token/revoke`

### 9.7 Usage
- `GET /api/v1/usage/summary`
- `GET /api/v1/usage/logs`

---

## 10) Security and Compliance

- Secrets (provider keys) must never be stored in plaintext.
- RLS is required when using Supabase.
- Rate limiting is required on shared/public endpoints.
- All artifact generation endpoints must be capability-gated.

---

## 11) Deployment Profiles (spec-level)

Deployment is defined by configuration, not code changes.

Profiles are enumerated in `05_BSNIO-Notebook-Research_CAPABILITIES_PROVIDERS_DEPLOYMENT.md`:
- `abacus_hosted`
- `netlify_frontend_vps_backend`
- `vps_monolith`
- `vercel_frontend_api_on_vps` (recommended alternative to “Vercel-only”)

---

## 12) Acceptance Criteria (V2 docs)

A V2-compliant implementation:
- Supports at least 3 LLM routes: Gemini direct, OpenAI direct, OpenAI-compatible (OpenRouter or RouteLLM)
- Supports at least 2 retrieval backends (Gemini File Search + pgvector)
- Exposes `/capabilities` and the frontend adapts UI accordingly
- Supports sharing notebooks + artifact share links
- Logs usage metrics (tokens/cost/storage)

