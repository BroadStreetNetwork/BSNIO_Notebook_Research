# BSNIO Notebook Research
## Implementation Guide (V2)

> **Version:** 2.0 (Docs Refresh) | **Date:** January 2026
>
> This guide is the **opinionated “paved road”** for implementing the V2 architecture using **Bun + TypeScript** (backend) and a decoupled frontend (typically Next.js). Everything is modular: provider adapters, retrieval backends, auth backends, storage backends, and deployment profiles.
>
> For matrices, env vars, and deploy checklists, see:
> - `05_BSNIO-Notebook-Research_CAPABILITIES_PROVIDERS_DEPLOYMENT.md`

---

## 0) What you are building

A source-centric research app with:
- Notebooks + Sources + Chat w/ citations
- Studio outputs (“Artifacts”) like reports, mind maps, slides, flashcards/quizzes, audio/video
- Sharing (notebook share links, artifact share links, shared chat)
- Provider abstraction (Gemini / Claude / OpenAI / OpenRouter / Abacus RouteLLM)
- Retrieval abstraction (Gemini File Search / Postgres+pgvector / Abacus Vector Store)
- Capability-driven UX

---

## 1) Choose a deployment profile first

Pick one profile and stick to it for the first implementation pass.

### Profile A: **Netlify Frontend + VPS Backend (recommended baseline)**
- Frontend: Netlify
- Backend API: VPS (Hostinger / Contabo / etc.) running Bun
- DB: Supabase or Postgres on VPS/managed
- Storage: S3-compatible (or Supabase Storage)

### Profile B: **Abacus-hosted App (Abacus deployment + RouteLLM)**
- App hosting and environment managed in Abacus
- RouteLLM for routing (OpenAI-compatible)
- Data: use Abacus-supported DB/auth (if available) OR connect external Postgres via connector; keep escape hatches
- Retrieval: Abacus Vector Store OR external pgvector

### Profile C: **VPS Monolith (frontend+backend behind one domain)**
- Nginx/Traefik reverse proxy
- Backend: Bun API
- Frontend: Next.js build served by Node/Bun
- DB/Storage: local or managed

> Vercel can still host the frontend, but Bun runtime support varies by platform. If Bun cannot run server-side in your chosen platform, keep the **backend on VPS or Abacus** and host the UI wherever you want.

---

## 2) Repo layout (recommended)

```
BSNIO_Notebook_Research/
  apps/
    api/                 # Bun backend
    web/                 # Next.js frontend (optional)
  packages/
    core/                # shared types, capability schema
    adapters/            # provider + retrieval + storage adapters
  docs/                  # planning docs
  scripts/               # local tooling
```

---

## 3) Backend (Bun) implementation

### 3.1 Initialize the Bun API app

```
cd apps
mkdir api && cd api
bun init -y
```

Recommended dependencies:
- HTTP: `hono` (fast, portable, small)
- Validation: `zod`
- DB: `postgres` (driver) + `drizzle-orm` (or Prisma if preferred)
- Auth: `jose` (JWT), optional `@supabase/supabase-js`
- Providers:
  - `openai` (OpenAI SDK, also works for OpenAI-compatible providers by setting `baseURL`)
  - `@anthropic-ai/sdk`
  - `@google/generative-ai` (or Google’s newer GenAI SDK if preferred)
- Storage:
  - `@aws-sdk/client-s3` (S3-compatible)
  - `@supabase/storage-js` (if using Supabase Storage)

```
bun add hono zod jose
bun add drizzle-orm postgres
bun add openai @anthropic-ai/sdk @google/generative-ai
bun add @aws-sdk/client-s3
```

### 3.2 Core concepts to implement (in order)

Implement the *interfaces first*, then the concrete adapters.

#### 3.2.1 Capabilities
- Implement `GET /api/v1/capabilities`
- Back it from env vars + adapter `health()` checks
- Use the canonical schema in doc 05

#### 3.2.2 Provider Abstraction Layer
Create adapters for:
- **LLM chat** (streaming preferred)
- **Embeddings** (optional but needed for pgvector)
- **Image** (optional)
- **TTS** (optional)
- **Video** (optional)

Minimum “day 1”:
- OpenAI-compatible adapter (OpenAI / OpenRouter / Abacus RouteLLM)
- Gemini adapter
- Claude adapter

#### 3.2.3 Retrieval Abstraction Layer
Implement `RagIndexProvider` with three backends:
- `gemini_file_search` (stores docs in Gemini store)
- `pgvector` (stores chunks and embeddings in Postgres)
- `abacus_vector_store` (stores docs in Abacus Vector Store)

Day 1 recommendation:
- Start with `gemini_file_search` OR `pgvector`.
- Add the third backend after baseline is stable.

#### 3.2.4 Data and storage abstraction
Implement:
- `DbAdapter` (Supabase Postgres vs direct Postgres)
- `StorageAdapter` (Supabase Storage vs S3 vs local)

---

## 4) Database: migrations + core tables

Implement the canonical schema from the V2 specification:
- `profiles`, `notebooks`, `sources`, `chat_sessions`, `chat_messages`
- `artifacts` (unified)
- `shares` (notebook / artifact / chat)
- `usage_events` (tokens/cost/storage)

### 4.1 pgvector mode
If `RAG_BACKEND=pgvector`:
- Ensure Postgres extension: `CREATE EXTENSION IF NOT EXISTS vector;`
- Add `chunks(embedding vector(N))` and (optional) `tsvector` columns.

---

## 5) Source ingestion

### 5.1 Source types
Implement ingestion for:
- `file` (pdf/docx/txt/md)
- `url` (web page)
- `youtube` (metadata + transcript)
- `text` (pasted)
- `repo` (GitHub repo snapshot / zip / local upload)

### 5.2 MCP ingestion (expansion)
Design ingestion as “connectors”:
- MCP servers can provide: Drive, GitHub, Notion, Slack, etc.
- The app ingests connector output as one or more `Source` records.

---

## 6) Chat + citations

### 6.1 Prompt contract
- System: “Answer from sources; cite; separate inference from cited facts.”
- Tooling: retrieval chunks injected + citations map

### 6.2 Multi-model chats
Support:
- Session default model
- Per-message override
- Optional: “compare models” mode (same prompt to multiple models)

Store:
- model used
- input/output tokens
- cost estimate

---

## 7) Studio outputs (Artifacts)

Implement a unified artifact pipeline:
- **Reports** (briefing, FAQ, study guide)
- **Flashcards / Quizzes**
- **Mind maps** (graph output + layout)
- **Slides** (PPTX)
- **Audio overviews** (script + TTS)
- **Video overviews** (script + generator)
- **Codebase outputs**: wiki, docs, how-to, “prompt pack”

Every artifact must:
- reference notebook + sources used
- store generation parameters
- store provider usage & cost
- be shareable via share links

---

## 8) Sharing

Implement:
- Public or restricted link sharing
- Viewer/editor roles
- Optional “view-only chat” for public notebooks

Share objects:
- Notebook share (notebook page + sources list + allowed artifacts)
- Artifact share (single artifact)
- Chat share (read-only transcript)

---

## 9) Frontend (Next.js) implementation

You can implement the UI with Next.js, but keep it decoupled:
- The UI reads `/capabilities` and adapts.
- Layout uses a “panel registry” rather than a hard-coded 3-panel design.

### 9.1 Layout engine
Implement a layout config like:
- `layout.mode = three_panel | two_panel | tabs | single`
- `layout.left = SourcesPanel | OutlinePanel | ...`
- `layout.right = StudioPanel | NotesPanel | ...`

---

## 10) Deployment

Use the checklists in doc 05.

### 10.1 VPS backend (Bun)
- Build + run as systemd service or Docker
- Put behind Nginx
- Configure env vars

### 10.2 Netlify frontend
- Set `NEXT_PUBLIC_API_URL`
- Build with Next.js output

### 10.3 Abacus deployment
- Configure RouteLLM base URL and key
- Configure DB/storage according to chosen profile

---

## 11) “Done means done” acceptance checks

Minimum acceptance:
- Create notebook
- Add sources (file + url + text)
- Chat with citations
- Generate one artifact (report)
- Share a notebook read-only
- Usage events recorded (tokens/cost)
- `/capabilities` accurately reflects configuration

