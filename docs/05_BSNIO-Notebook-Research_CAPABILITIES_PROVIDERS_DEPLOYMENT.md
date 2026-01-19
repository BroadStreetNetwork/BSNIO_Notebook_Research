# BSNIO Notebook Research
## Capabilities, Providers, Environment Variables, Deployment Profiles (V2)

> **Purpose:** This document is the single reference for **what the app can do**, **which providers/backends are enabled**, **how to configure it** (env vars), and **how to deploy it** across multiple platforms.
>
> This doc complements:
> - `01_BSNIO-Notebook-Research_VISION_DOCUMENT.md`
> - `02_BSNIO-Notebook-Research-PROJECT_SPECIFICATION.md`
> - `03_BSNIO-Notebook-Research-IMPLEMENTATION_GUIDE.md`
> - `04_BSNIO-Notebook-Research_DOCS_UPDATE_PLAN_2026_v2.md`

---

## 1) Capability-Driven Product (the rule)

Everything is **capability-gated**:

- If a provider isn’t configured → the endpoint returns a clear error and the UI hides/disables the feature.
- If a storage backend isn’t configured → uploads are disabled (but “text-only notebooks” can still work).
- If a retrieval backend isn’t configured → chat works in “no-RAG mode” (plain chat) but citations are disabled.

### 1.1 Canonical Capabilities Schema

The backend must expose:
- `GET /api/v1/capabilities` (safe-to-share subset)
- `GET /api/v1/admin/capabilities` (full details)

**Canonical schema (example):**
```json
{
  "app": {
    "name": "BSNIO Notebook Research",
    "version": "2.0.0",
    "build": "2026-01-19",
    "environment": "production"
  },
  "data": {
    "db": { "enabled": true, "type": "postgres", "provider": "supabase", "migrations_ok": true },
    "storage": { "enabled": true, "type": "s3", "provider": "supabase_storage" },
    "cache": { "enabled": false, "type": null },
    "telemetry": { "enabled": true }
  },
  "auth": {
    "mode": "hybrid",
    "identity": {
      "enabled": true,
      "providers": ["supabase", "oauth_google"],
      "allow_email_password": true
    },
    "api_keys": {
      "enabled": true,
      "scopes": true,
      "rate_limits": true
    },
    "sharing": {
      "enabled": true,
      "modes": ["invite", "link"],
      "permissions": ["viewer", "editor"],
      "allow_public_read": false
    }
  },
  "providers": {
    "llm": {
      "enabled": true,
      "default": { "provider": "gemini", "model": "gemini-2.5-flash" },
      "available": [
        { "provider": "gemini", "models": ["gemini-2.5-flash", "gemini-2.5-pro"] },
        { "provider": "openai", "models": ["gpt-4o-mini", "gpt-4.1"] },
        { "provider": "anthropic", "models": ["claude-3.7-sonnet"] },
        { "provider": "openrouter", "models": ["openai/gpt-4o", "anthropic/claude-3.7-sonnet"] },
        { "provider": "abacus_routellm", "models": ["route-llm"] }
      ]
    },
    "embeddings": {
      "enabled": true,
      "default": { "provider": "openai", "model": "text-embedding-3-small" }
    },
    "rerank": { "enabled": false },
    "image": { "enabled": true, "default": { "provider": "gemini", "model": "imagen" } },
    "tts": { "enabled": true, "default": { "provider": "openai", "model": "gpt-4o-mini-tts" } },
    "video": { "enabled": false, "default": null },
    "web_search": { "enabled": true, "provider": "builtin" }
  },
  "rag": {
    "enabled": true,
    "mode": "pgvector",
    "modes_available": ["gemini_file_search", "pgvector", "abacus_vector_store"],
    "citations": true,
    "chunking": { "enabled": true, "max_chunk_chars": 2000 },
    "limits": { "max_sources_per_notebook": 500, "max_context_tokens": 200000 }
  },
  "jobs": {
    "mode": "postgres_queue",
    "enabled": true,
    "supports": ["ingest", "audio", "video", "exports", "deep_research"],
    "realtime_progress": true
  },
  "features": {
    "notebooks": true,
    "sources": true,
    "chat": true,
    "multi_model_chat": true,
    "mind_map": true,
    "reports": true,
    "slides": true,
    "flashcards": true,
    "quizzes": true,
    "audio_overview": true,
    "video_overview": false,
    "codebase_wiki": true,
    "how_to_docs": true,
    "prompt_packs": true
  },
  "limits": {
    "max_upload_mb": 200,
    "max_notebooks_per_user": 200,
    "max_chat_history_messages": 2000
  }
}
```

---

## 2) Provider Compatibility Matrix (by capability)

This is the **adapter matrix**. The platform supports these providers *via* a Provider Abstraction Layer. In practice, your deployment may enable only a subset.

### 2.1 LLM (chat / reasoning)

| Provider | API style | Notes |
|---|---|---|
| Google Gemini | Native | Strong multimodal, pairs well with Gemini File Search. |
| Anthropic Claude | Native | Strong long-context reasoning; good for codebase/wiki generation. |
| OpenAI | Native | Broad ecosystem; strong tools support. |
| OpenRouter | OpenAI-compatible | Routes many models through one endpoint. |
| Abacus RouteLLM | OpenAI-compatible | “Smart routing” (cost/speed/perf) through a single model id. |

**Multi-model chats:** supported per message turn (choose provider/model each turn), plus optional “router mode” to auto-select.

### 2.2 Embeddings (for pgvector / Abacus vector store)

| Provider | Notes |
|---|---|
| OpenAI | Default paved road for embeddings. |
| Gemini | Optional; depends on chosen embedding interface. |
| OpenRouter | Use embedding-capable models via OpenAI-compatible calls. |
| Abacus | If using Abacus Vector Store, embeddings can be handled internally (platform-managed) or externally via your adapter. |

### 2.3 Rerank (optional quality boost)

| Provider | Notes |
|---|---|
| (Optional) Cohere / other | Add later. Keep interface ready, do not hard-require for MVP. |

### 2.4 Image generation (first-class artifact)

| Provider | Notes |
|---|---|
| Gemini (Imagen via Gemini API) | Default image path when Gemini is enabled. |
| OpenAI Images | Common alternative. |
| OpenRouter | Some image-capable models are accessible via their unified API; implement via adapter. |

### 2.5 Text-to-Speech (TTS)

| Provider | Notes |
|---|---|
| Gemini TTS | Works when Gemini TTS is enabled. |
| OpenAI TTS | Common alternative for high quality voices. |
| ElevenLabs (optional) | Add as adapter if desired. |

### 2.6 Video generation

| Provider | Notes |
|---|---|
| AtlasCloud (Wan 2.5) | Keep as one option, not the only option. |
| (Optional) Replicate / Runway / others | Add via adapter later; keep interface stable. |

---

## 3) Retrieval / RAG Backend Matrix

RAG is **pluggable**. Choose one mode per deployment (or per notebook if you want to get fancy later).

| Mode | Best for | Pros | Cons |
|---|---|---|---|
| `gemini_file_search` | Lowest infra ops | No vector DB to manage; citations often “built-in” | Vendor dependency; some control tradeoffs |
| `pgvector` | “Open stack” deployments | Transparent, portable, controllable; supports keyword + metadata filters | You manage chunking/embeddings/reindexing |
| `abacus_vector_store` | Abacus-hosted ecosystems | Managed ingestion and retrieval; integrates with Abacus stack | External dependency; align with Abacus retriever semantics |

**Keyword + metadata:**
- `pgvector`: use `tsvector` (keyword) + JSONB metadata filters + pgvector similarity.
- `gemini_file_search`: use provider-native metadata filtering.
- `abacus_vector_store`: use provider-native filters.

---

## 4) Environment Variables Reference (V2)

### 4.1 Core App
```bash
APP_NAME="BSNIO Notebook Research"
APP_ENV=development|staging|production
APP_BASE_URL=https://yourdomain.com
API_BASE_URL=https://api.yourdomain.com
LOG_LEVEL=info

# Jobs
JOBS_MODE=inline|postgres_queue

# Capabilities
CAPABILITIES_PUBLIC_SAFE=true
```

### 4.2 Auth
```bash
AUTH_MODE=disabled|jwt_only|supabase|oidc|hybrid

# Supabase identity (optional)
SUPABASE_URL=
SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# OIDC (optional)
OIDC_ISSUER_URL=
OIDC_CLIENT_ID=
OIDC_CLIENT_SECRET=
OIDC_REDIRECT_URL=

# API keys
API_KEYS_ENABLED=true
API_KEYS_HASH_SECRET=
```

### 4.3 Database
```bash
DB_PROVIDER=supabase_postgres|postgres|abacus_managed
DB_URL=postgresql://user:pass@host:5432/db
DB_SSLMODE=require
DB_MIGRATIONS_PATH=./migrations
```

### 4.4 Storage
```bash
STORAGE_PROVIDER=supabase_storage|s3|local_fs|abacus_file_hosting

# S3-compatible
S3_ENDPOINT=
S3_BUCKET=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
S3_REGION=
S3_PUBLIC_BASE_URL=

# Local FS (VPS only)
LOCAL_STORAGE_PATH=/var/lib/bsnio-notebook-research
```

### 4.5 RAG / Retrieval
```bash
RAG_MODE=none|gemini_file_search|pgvector|abacus_vector_store

# pgvector + app-managed RAG
EMBED_PROVIDER=openai|gemini|openrouter
EMBED_MODEL=text-embedding-3-small
PGVECTOR_DIM=1536
CHUNK_MAX_CHARS=2000
CHUNK_OVERLAP_CHARS=200

# Gemini File Search
GEMINI_FILE_SEARCH_PROJECT=
GEMINI_FILE_SEARCH_LOCATION=

# Abacus Vector Store
ABACUS_VECTOR_STORE_ID=
ABACUS_API_KEY=
```

### 4.6 Providers (LLM + modalities)

#### Gemini
```bash
GEMINI_API_KEY=
GEMINI_DEFAULT_MODEL=gemini-2.5-flash
GEMINI_ENABLE_TTS=true
GEMINI_ENABLE_IMAGE=true
```

#### Anthropic
```bash
ANTHROPIC_API_KEY=
ANTHROPIC_DEFAULT_MODEL=claude-3.7-sonnet
```

#### OpenAI
```bash
OPENAI_API_KEY=
OPENAI_DEFAULT_MODEL=gpt-4o-mini
OPENAI_ENABLE_TTS=true
OPENAI_ENABLE_IMAGE=true
```

#### OpenRouter
```bash
OPENROUTER_API_KEY=
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
OPENROUTER_DEFAULT_MODEL=openai/gpt-4o-mini
```

#### Abacus RouteLLM
```bash
ABACUS_ROUTELLM_API_KEY=
ABACUS_ROUTELLM_BASE_URL=https://routellm.abacus.ai/v1
ABACUS_ROUTELLM_MODEL=route-llm
```

#### Video providers
```bash
VIDEO_PROVIDER=none|atlascloud|replicate|...
ATLASCLOUD_API_KEY=
```

---

## 5) Deployment Profiles (checklists)

A “Deployment Profile” is a **documented, supported combo** of:
- Runtime (where the API runs)
- Auth choice
- DB choice
- Storage choice
- RAG mode

### 5.1 Profile A: Abacus-hosted (API + optional UI)

**Use when:** you want Abacus-managed app hosting plus Abacus ecosystem integrations.

Checklist:
- [ ] Configure Abacus-hosted app runtime secrets for `ABACUS_ROUTELLM_API_KEY` (and/or other provider keys)
- [ ] Choose DB: Abacus-managed DB (if available) *or* external Postgres via connector
- [ ] Choose storage: Abacus file hosting (if used) *or* external S3/Supabase Storage
- [ ] Choose RAG: `abacus_vector_store` or `gemini_file_search` or `pgvector`
- [ ] Enable `GET /api/v1/capabilities` and verify flags
- [ ] Confirm custom domain routing (API + UI)

### 5.2 Profile B: Netlify (UI) + VPS (API)

Checklist:
- [ ] Deploy UI to Netlify with `NEXT_PUBLIC_API_BASE_URL`
- [ ] Deploy API to VPS (Bun) behind Nginx/Traefik
- [ ] Choose DB: Postgres (managed or VPS) + optional pgvector
- [ ] Choose storage: S3-compatible or local FS
- [ ] Choose auth: OIDC or JWT-only, optionally Supabase Auth

### 5.3 Profile C: VPS Monolith (UI + API)

Checklist:
- [ ] Single domain: `/` → UI, `/api` → API
- [ ] Nginx: static UI + reverse proxy to Bun API
- [ ] Postgres + pgvector locally or managed
- [ ] S3-compatible storage recommended; local FS acceptable

### 5.4 Profile D: “Vercel-ish UI” + external API

Use Vercel for UI, and keep API on VPS/Abacus. Avoid fighting runtime constraints.

Checklist:
- [ ] Vercel UI deployed
- [ ] API deployed elsewhere
- [ ] CORS configured

---

## 6) Git-style Diff Summary + PR Checklist (Docs Refresh)

### 6.1 Diffstat (conceptual)

```
 docs/01_BSNIO-Notebook-Research_VISION_DOCUMENT.md              | rewritten for V2 abstractions + capabilities
 docs/02_BSNIO-Notebook-Research-PROJECT_SPECIFICATION.md        | rewritten for Bun/TS + pluggable RAG + sharing + artifacts
 docs/03_BSNIO-Notebook-Research-IMPLEMENTATION_GUIDE.md          | rewritten for Bun/TS build + deployment profiles
 docs/04_BSNIO-Notebook-Research_DOCS_UPDATE_PLAN_2026_v2.md       | retained as the execution roadmap
 docs/05_BSNIO-Notebook-Research_CAPABILITIES_PROVIDERS_DEPLOYMENT.md | NEW reference doc
 README.md                                                       | NEW/UPDATED repository entrypoint
```

### 6.2 PR checklist

Core alignment:
- [ ] All docs reference **Capabilities** and **Deployment Profiles** consistently
- [ ] No hard-coded app name; uses `APP_NAME`
- [ ] No hard-coded static LLM pricing tables in the logic; pricing is configurable and updateable
- [ ] Supabase is not the only DB/auth/storage choice
- [ ] Vercel is not the only deployment target

Providers:
- [ ] Gemini, Anthropic, OpenAI, OpenRouter, Abacus RouteLLM are documented and adapter-ready
- [ ] Image generation is first-class and capability-gated
- [ ] Video/TTS are provider-pluggable (no single-vendor lock)

Retrieval:
- [ ] RAG supports `gemini_file_search`, `pgvector`, and `abacus_vector_store`
- [ ] Keyword/metadata filtering is documented for each retrieval mode

Sharing + artifacts:
- [ ] Notebook sharing, source sharing, chat sharing, artifact sharing are documented
- [ ] Artifact system covers audio/video/slides/flashcards/quizzes/mind maps/reports/wiki/how-to/prompt packs

Observability:
- [ ] Token usage, cost, storage bytes, context-window metrics, and logs are specified

Security:
- [ ] Auth abstraction (Supabase/OIDC/JWT-only) is documented
- [ ] API-key authorization scopes are documented

---

## 7) “Setup Checklist” UI (what the user sees)

The UI should have a **Setup / Diagnostics** screen that:
- calls `/api/v1/capabilities`
- shows missing config
- offers copy-paste env var examples
- confirms RAG mode health
- confirms storage write test

---

**End of reference.**
