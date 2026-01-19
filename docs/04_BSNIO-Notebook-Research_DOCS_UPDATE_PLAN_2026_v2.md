# BSNIO Notebook Research (NotebookLM Clone)
## 2026 Documentation Refresh + Bun Refactor Update Plan

This file is a **precise, execution-oriented plan** to update the existing planning docs:

- `01_BSNIO-Notebook-Research_VISION_DOCUMENT.md`
- `02_BSNIO-Notebook-Research-PROJECT_SPECIFICATION.md`
- `03_BSNIO-Notebook-Research-IMPLEMENTATION_GUIDE.md`

…so they reflect a **Bun/TypeScript refactor**, **provider/data/auth/deploy abstraction layers**, **capabilities introspection**, **image generation**, **MCP-based ingestion expansion**, and **new “put learning to work” outputs** (codebase understanding, wikis/docs/how-to, prompt generation).

---

## 0) North Star Decisions (Lock These into the Docs)

### 0.1 Primary Product Thesis (updated)
- Keep the original “API-first, open platform” thesis.
- Replace “Supabase for everything” with “**Paved roads + escape hatches**”:
  - Supabase remains a first-class default.
  - Postgres/S3/OIDC become first-class alternatives.
  - Abacus ecosystem (App Hosting + Database + File Storage + Vector Store + RouteLLM) becomes a first-class option.

### 0.2 Core Refactor Decision: Bun-first backend
- Replace FastAPI/Python backend sections with a **Bun + TypeScript backend**.
- Keep Next.js (or equivalent) frontend, but decouple it from backend deployment.
- Target three primary deployment profiles:
  1) **Decoupled**: Frontend (Netlify) + Backend API (VPS/Abacus)
  2) **Vercel Fullstack**: Next.js + Bun runtime for functions
  3) **VPS Monolith**: Backend + frontend behind Nginx/Traefik

### 0.3 Non-goal for this refresh
- Local desktop/Bun runtime packaging is **out of scope** unless it later simplifies web deployment.

---

## 1) New Abstraction Layers (Add to All Docs as First-Class Concepts)

### 1.1 Provider Abstraction Layer (Generative AI)
**Goal:** support BYO keys and routing for:
- Google Gemini (direct)
- Anthropic Claude (direct)
- OpenAI (direct)
- OpenRouter (OpenAI-compatible)
- Abacus RouteLLM (OpenAI-compatible)

**Required provider categories (expand beyond LLM):**
- LLM (chat/completions)
- Embeddings
- Rerank (optional but recommended)
- Image Generation (new)
- TTS
- Video
- Web Search / Browsing (optional; can be provider-native or external)

**Doc requirement:** every provider is “plug-in” via adapters; features are gated by **Capabilities**.

### 1.2 Data Abstraction Layer (Database + Storage)
- Database adapters:
  - Supabase Postgres (default)
  - Direct Postgres (VPS/managed)
  - Abacus App Database (Abacus-hosted deployments)
  - Optional later: SQLite (only if local returns)
  - **Note:** pgvector is supported via Postgres extension; enable it when `retrieval.mode = pgvector`.
- Storage adapters:
  - Supabase Storage (default)
  - S3-compatible (R2/S3/Wasabi/etc.)
  - Abacus File Storage (Abacus-hosted deployments; Abacus-managed file hosting / SFTP connector where applicable)
  - Local filesystem (VPS-only)

### 1.3 Auth Abstraction Layer
Two-tier architecture:
- **Identity** (how users sign in): Supabase Auth, OAuth (Google/GitHub), OIDC
- **Authorization** (what they can access): JWT verification, API keys, roles/permissions for sharing

### 1.4 Deployment Abstraction Layer (as “Deployment Profiles”)
Avoid runtime complexity. Keep it as:
- A small set of **deployment profiles**
- Each profile defines required env vars, storage/db/auth choices, and job execution patterns

---

## 2) Capabilities: Add a Self-Aware Configuration System

### 2.1 Capabilities Manifest (must be in docs + code)
**Define a canonical `Capabilities` object** computed at runtime from env/config:
- `db`: configured? type? migrations OK?
- `storage`: configured? type?
- `auth`: configured? identity providers enabled?
- `providers`: which are enabled (llm/embeddings/rerank/image/tts/video/search)
- `rag`: retrieval/index configured? which backend? (gemini_file_search / pgvector / abacus_vector_store)
- `features`: which product features are enabled (research, sharing, artifacts, codebase tools, etc.)
- `jobs`: background jobs available? queue mode?
- `limits`: max file size, rate limits, token limits, storage quotas

### 2.2 Required endpoints + UI surfaces
- `GET /api/v1/capabilities` (public safe subset)
- `GET /api/v1/admin/capabilities` (full detail)
- Admin UI: “Setup Checklist” that reads capabilities and shows missing config

### 2.3 Documentation requirement
Every feature section must include:
- **Capability flag name**
- Required providers + minimum requirements
- Degraded mode behavior (e.g., “no video provider => hide video UI + reject endpoint gracefully”)

---

## 3) RAG / Retrieval + Vector Index: Make It Pluggable

The original docs assume **Gemini File Search** as the retrieval layer. That’s valid (and often cost-effective), but the refreshed docs must support **multiple, swappable RAG backends**.

### 3.1 Introduce a RAG Index Abstraction (first-class)
Define a `RagIndexProvider` interface with:
- `ingest(source) -> external_doc_id`
- `delete(external_doc_id)`
- `query(query, filters) -> {chunks, scores, citations?}`
- `health()`

### 3.2 Supported RAG backends (document all three)
- **Backend A: Postgres + pgvector (App-managed RAG)**
  - Chunking + embeddings + similarity search in Postgres
  - Works on Supabase Postgres *or* any Postgres you manage
- **Backend B: Gemini File Search (Provider-native RAG)**
  - Low-ops path when Gemini is acceptable for retrieval
  - Supports semantic retrieval + metadata filters + grounding/citations
- **Backend C: Abacus Vector Store (Managed vector DB)**
  - Use Abacus Vector Store for embedding storage + retrieval when operating in the Abacus ecosystem

### 3.3 Retrieval modes and routing rules
Add `retrieval.mode` (capability + config):
- `pgvector` | `gemini_file_search` | `abacus_vector_store` | `off`

Add `retrieval.policy`:
- `always` (must retrieve)
- `auto` (retrieve when query likely needs sources)
- `never` (chat-only)

### 3.4 Doc changes required
- Add `rag` capability + config
- Add embedding provider selection (Gemini / Anthropic / OpenAI / OpenAI-compatible via OpenRouter/RouteLLM)
- Add ingestion status tracking per source (indexed / failed / pending)
- Add metadata/keyword filters support in the retrieval API (even if backend is semantic)

## 4) Image Generation (New, Missing Today)

### 4.1 Add Image Generation as a first-class provider category
- `ImageProvider.generate()` (text-to-image)
- Optional: `ImageProvider.edit()` (inpainting/outpainting)

### 4.2 Add image outputs to Artifacts
- “Concept image” for a notebook
- “Diagram” or “illustration” for a section
- “Cover image” for audio/video
- “Study cards” visuals (optional)

### 4.3 Capability flags
- `providers.image.enabled`
- `features.artifacts.image`

---

## 5) MCP Expansion (Inputs + Tooling)

### 5.1 Define MCP role in this project
MCP servers become **source ingestion connectors** and/or **tooling connectors**.

Two patterns:
- **Ingestion connectors**: pull content into a notebook as sources (GitHub repo/docs, Notion pages, Google Drive files, Confluence, etc.)
- **Action connectors**: optionally enable “write-back” workflows (create issues, update docs). Keep this off by default.

### 5.2 MCP security posture (must be documented)
- Explicit allowlist of MCP servers per deployment
- Per-user credential vault rules
- Prompt-injection protections: treat external tool output as untrusted; enforce tool schemas; log tool calls

### 5.3 Required doc updates
- Add an “MCP Connectors” section:
  - Supported connectors for v1 (minimum): GitHub, Notion, Google Drive
  - Expand list as optional: Slack, Confluence, Linear/Jira
- Add ingestion endpoints:
  - `POST /sources/mcp` (create sources from MCP resources)
  - `POST /mcp/servers` (admin: register/allow a server)

---

## 6) New Output Features (“Put learning to work”)

Add a new product pillar: **Actionable Knowledge Outputs**.

### 6.1 Codebase Understanding
New feature bundle:
- Ingest a repo (zip upload, Git clone on VPS, or MCP GitHub)
- Parse structure + key files + dependencies
- Create:
  - Architecture overview
  - Component map
  - Data flow map
  - “Where to change X” guides

### 6.2 Wiki / Documentation / How-to Generation
New artifact types:
- Wiki pages (MD)
- Developer docs (MD)
- End-user how-to (MD/PDF)
- Runbooks (MD)
- API reference stubs (MD)

### 6.3 Prompt / Instruction Generation Studio
New artifact types:
- “System prompt” drafts
- “Instruction templates”
- “Automation prompts” (n8n/Make)
- “Evaluation prompts” (rubrics / checkers)

### 6.4 Document requirements
- Add these features to “Beyond NotebookLM” section
- Add endpoints under `/artifacts` (recommended) rather than one-off endpoints

---

## 7) Schema Updates (Spec must be edited precisely)

### 7.1 New/changed tables
Add (or extend existing) tables to support provider-agnostic operation:

1) `rag_indexes`
- rag_index_id
- provider_id (pgvector | gemini_file_search | abacus_vector_store)
- config (json)
- created_at / updated_at

2) `rag_documents`
- rag_index_id
- source_id
- external_doc_id
- status (pending|indexed|failed)
- metadata (json)
- created_at / updated_at

3) `model_registry`
- provider_id
- model_id
- capabilities (json)
- pricing (json)
- context_window
- updated_at

4) `provider_credentials`
- owner_type (user/workspace)
- owner_id
- provider_id
- credential_type (api_key/oauth)
- encrypted_secret_ref
- created_at / rotated_at

5) `source_chunks`
- source_id
- chunk_index
- text
- embedding (vector) (ONLY when using pgvector backend)
- token_count

6) `artifacts`
- notebook_id
- artifact_type (audio/video/image/wiki/howto/prompt/slides/flashcards/table/etc.)
- status
- provider_used
- model_used
- storage_path
- metadata (json)
- usage (json)

7) `share_links` and/or `resource_permissions`
- share tokens + permissions

### 7.2 Existing tables to extend
- `chat_messages`: add optional multi-model fields (router decisions, fallback model)
- `usage_logs`: store provider, latency, context-window metrics, retrieval stats

### 7.3 RLS notes
- Document RLS for sharing (read-only links, workspace roles)

---

## 8) API Spec Updates (Make it provider-agnostic)

### 8.1 Replace single-provider assumptions
- Remove Gemini-only endpoints and rename to provider-agnostic:
  - `/chat` remains, but accepts `{ provider, model }` or `{ route: "auto" }`
  - `/audio`, `/video`, `/image` become artifact requests

### 8.2 Add key endpoints
- `GET /providers` (enabled providers)
- `GET /models` (from registry; filtered by capabilities)
- `POST /credentials` (store BYO keys; user or workspace scoped)
- `GET /capabilities`
- `POST /artifacts` (generic)
- `GET /artifacts/:id`
- `POST /sources/mcp`

### 8.3 Multi-model chat
- Allow per-message model override
- Allow router mode:
  - `{ route: "abacus" }` (RouteLLM)
  - `{ route: "openrouter" }`
  - `{ route: "manual" }`

---

## 9) Costing + Metrics (Remove hardcoded tables)

### 9.1 Replace “Model Pricing Reference” sections
- Move static pricing tables out of the docs.
- Replace with:
  - `model_registry` + update mechanism
  - “cost computation rules”

### 9.2 Add required metrics
- token usage (in/out)
- retrieval tokens + sources used
- context window total/used
- storage bytes by notebook/user
- latency (TTFT + total)
- provider failure rate

---

## 10) Deployment Profiles (Docs must become multi-platform)

### 10.1 Add a “Deployment Profiles” section to Vision + Spec + Implementation

**Profile A: Frontend Netlify + Backend VPS/Abacus**
- Frontend: Next.js on Netlify
- Backend: Bun API on VPS **or Abacus-hosted app runtime**
- Storage/DB options:
  - Supabase (Postgres + Storage)
  - Direct Postgres (+pgvector) + S3/R2
  - **Abacus App Database + Abacus File Storage + Abacus Vector Store** (Abacus ecosystem)

**Profile B: Vercel Fullstack (Bun Functions)**
- Next.js + Bun runtime for API routes/functions
- DB/Storage: Supabase default; alternatives allowed

**Profile C: VPS Monolith**
- Reverse proxy (Nginx/Traefik)
- Bun API service + frontend SSR/static
- Local FS storage allowed

### 10.2 Add a `deploy/` directory spec
Document required recipe folders:
- `deploy/vps-docker/`
- `deploy/vps-baremetal/`
- `deploy/netlify/`
- `deploy/vercel/`
- `deploy/abacus/`

Abacus recipe must include (in addition to the standard checklist):
- RouteLLM configuration (optional router)
- Abacus App Database + File Storage selection
- Abacus Vector Store and/or Gemini File Search selection
- Abacus Secrets / key management strategy

Each recipe must include:
- env var checklist
- secrets checklist
- build & release commands
- background job strategy
- health checks

---

## 11) Bun Refactor Guidance (Spec + Implementation Guide)

### 11.1 Replace backend implementation sections
- Remove Python/FastAPI examples.
- Replace with:
  - Bun server framework choice (documented as an implementation detail)
  - route structure mirroring existing endpoints
  - typed DTOs + validation
  - streaming support (SSE)
  - provider adapters

### 11.2 Monorepo structure (recommended)
Document as:
- `apps/api` (Bun)
- `apps/web` (Next.js)
- `packages/core` (shared types, provider interfaces, capabilities)
- `packages/providers/*` (gemini, anthropic, openai-compatible)
- `packages/connectors/*` (mcp connectors)

### 11.3 Update “AI-assisted development” principle
- Expand from “Supabase MCP only” to “MCP suite” (Supabase + GitHub/Notion/etc.)

---

## 12) Precise Edit Instructions Per Existing Document

### 12.1 Vision Document (`01_..._VISION_DOCUMENT.md`)
**Make these edits:**
1) Update prerequisites: remove “Supabase + Vercel + Gemini required” framing.
2) Replace “SUPABASE FOR EVERYTHING” principle with “Paved roads + escape hatches”.
3) Replace architecture diagram: show Bun API + provider adapters + data/auth adapters.
4) Replace “Gemini Models Reference” section with:
   - “Provider + Model Registry” concept
   - Example of switching provider/model
5) Add new feature bullets:
   - Image generation artifacts
   - MCP ingestion connectors
   - Codebase understanding + wiki/how-to/prompt studio outputs
6) Add “Capabilities” section:
   - explain introspection + degraded modes

### 12.2 Project Specification (`02_..._PROJECT_SPECIFICATION.md`)
**Make these edits:**
1) Rewrite prerequisites:
   - Add provider key options list (Gemini/Anthropic/OpenAI/OpenRouter/RouteLLM)
   - Add deployment profiles
2) Replace sections 5–8 (Gemini integration / AtlasCloud / FastAPI) with:
   - Provider abstraction layer spec
   - Image provider spec
   - RAG/retrieval spec (pluggable: pgvector / Gemini File Search / Abacus Vector Store)
   - Bun API spec (routes + streaming)
3) Replace “Model Pricing Reference” with model registry + costing rules.
4) Update database schema:
   - add tables from Section 7
   - update usage metrics fields
5) Expand API endpoints:
   - `/capabilities`, `/providers`, `/models`, `/credentials`, `/artifacts`, `/sources/mcp`
6) Update frontend spec:
   - make layout configurable (no hardcoded three-panel)
   - add setup checklist UI driven by capabilities
   - add multi-model selection + routing UI
7) Replace deployment section:
   - add multi-profile instructions + `deploy/` recipes

### 12.3 Implementation Guide (`03_..._IMPLEMENTATION_GUIDE.md`)
**Make these edits:**
1) Replace Phase 2 (Backend) with Bun build steps:
   - monorepo setup
   - capabilities + config validation
   - provider interfaces + adapters
   - RAG pipeline
   - artifacts system
2) Replace “Use Supabase MCP for everything” with:
   - “Use MCP where it helps” (Supabase + connectors)
3) Add Phase: “Deployment Profiles”
   - steps for Netlify+VPS, Vercel fullstack, VPS monolith
4) Update test commands and verification checklist:
   - validate capabilities endpoint
   - validate multi-provider keys
   - validate artifact generation across providers
   - validate MCP ingestion (GitHub/Notion/Drive)

---

## 13) Optional: Add One New Doc (Recommended)

To avoid bloating the three core docs, add a new doc:

- `05_CAPABILITIES_PROVIDERS_DEPLOYMENT.md`

It holds:
- provider matrices
- capabilities schema
- deployment profiles
- env var reference

This keeps Vision clean and Spec readable.

---

## 14) Acceptance Criteria (Docs “Done” Definition)

Docs are considered updated when:
- No section assumes a single provider (Gemini-only removed)
- No section assumes a single infra vendor (Supabase-only removed; Abacus + VPS-native supported)
- Deployment section covers at least: Abacus, Netlify+VPS, Vercel
- Capabilities are defined and referenced everywhere
- Image generation is included as a provider + artifact type
- MCP ingestion connectors are specified and secured
- New outputs (codebase/wiki/how-to/prompt studio) are specified as artifacts + endpoints
- Pricing is no longer hardcoded; model registry is the source of truth

---

## Appendix A: Reference Links (keep URLs out of primary doc text)

```text
Netlify Bun in builds:
- https://www.netlify.com/blog/bun-support-in-builds/

Vercel Bun runtime docs:
- https://vercel.com/docs/functions/runtimes/bun
- https://vercel.com/blog/bun-runtime-on-vercel-functions

MCP servers index:
- https://github.com/modelcontextprotocol/servers
- https://github.com/appcypher/awesome-mcp-servers

RAG backends:
- https://codelabs.developers.google.com/gemini-file-search-for-rag#0
- https://github.com/pgvector/pgvector
- https://abacus.ai/vectorstore

Abacus app infra pricing (hosting, database, file storage):
- https://abacus.ai/help/chatllm-ai-super-assistant/pricing-apps

Abacus managed SFTP storage connector:
- https://abacus.ai/help/connectors/fileConnectors/sftp
```
