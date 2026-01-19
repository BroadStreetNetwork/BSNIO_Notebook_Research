# BSNIO Notebook Research — Vision (V2.1)

## Purpose
Build an open, multi-provider “NotebookLM-style” research workspace that turns **sources → grounded understanding → reusable outputs**.

This is not just a chat UI. It’s a system for:
- ingesting many source types (files, URLs, repos, docs via MCP)
- retrieving relevant evidence (semantic + metadata filtering)
- producing grounded responses with citations
- generating reusable outputs (wikis, docs, how-tos, prompt packs, study artifacts)
- deploying across environments (Abacus AI hosted, Netlify, private VPS; Vercel optional)

## Non-negotiables
1) **Provider independence**: LLMs, TTS, image/video generation, and retrieval/search are swappable via adapters.
2) **Data plane independence**: Relational DB, object storage, and vector/RAG index are separate decisions.
3) **Deployment independence**: Deployments are “profiles” (recipes), not hard-coded assumptions.
4) **Capability introspection**: The app is self-aware and degrades safely when a capability isn’t configured.

## Core abstraction layers

### A) Provider layer
Support BYO keys per user or per workspace.

**LLMs**
- Google Gemini
- Anthropic Claude
- OpenAI
- OpenRouter (OpenAI-compatible)
- Abacus RouteLLM (OpenAI-compatible router)

**Media/artifact providers**
- TTS: Gemini / OpenAI / others via adapters
- Image generation: Gemini Imagen / OpenAI Images / OpenRouter image-capable models
- Video: AtlasCloud (existing) + additional providers via adapters (Abacus tools where available)

**Multi-model chat is a first-class feature**
- pick a model per message
- route automatically (RouteLLM)
- draft + verify (cheap model → verification model)

### B) Data plane layer (3 independent choices)
We explicitly separate **Relational DB** and **Vector/RAG Index** so they can be mixed and matched.

#### 1) Relational DB (metadata + transactions)
- Postgres (self-hosted) / Supabase Postgres
- MySQL
- SQLite (embedded)
- Turso (hosted SQLite)
- DuckDB (embedded analytics; best for analysis-heavy notebooks/offline modes)
- Abacus transactional DB (optional adapter)

#### 2) Object storage (files + artifacts)
- Supabase Storage
- S3-compatible (AWS S3, Cloudflare R2, etc.)
- Abacus storage (adapter)
- Local filesystem (VPS/dev)

#### 3) Vector store / RAG index (retrieval)
- Gemini File Search ("Gemini File Store") — managed chunk+embed+semantic retrieval with metadata filters
- Postgres + pgvector — app-managed embeddings + similarity search
- SQLite/Turso + sqlite-vector — embedded similarity search
- Qdrant — dedicated vector DB
- LanceDB — embedded/local-first vector DB
- Abacus Vector Store — managed (adapter)
- vectorwrap — optional wrapper to unify multiple vector backends

**Compatibility constraints (enforced by config validation + capabilities)**
- pgvector requires Postgres (including Supabase Postgres)
- sqlite-vector requires SQLite or Turso

**Examples we explicitly support**
- MySQL + Gemini File Store (cheap, portable)
- SQLite + Gemini File Store (simple metadata + managed retrieval)
- SQLite/Turso + sqlite-vector (single-service simplicity)
- DuckDB + Qdrant/LanceDB (analytics + retrieval)
- Postgres + pgvector (tightest integrated DB story)

### C) Auth layer
- Supabase Auth
- OAuth/OIDC (Google, GitHub)
- JWT-only / SSO-forwarded headers for private deployments
- Dev/no-auth profile (local/dev only)

### D) Deployment profiles
- Abacus-hosted
- VPS (Docker Compose recommended)
- Netlify web + external API/worker
- Vercel (optional)

## Product pillars (outputs)
Beyond “chat with sources,” the system must quickly produce outputs that help teams execute.

- **Codebase understanding**: ingest repos, generate architecture maps, API inventories, dependency diagrams (text/mermaid), change guidance grounded in code citations.
- **Wikis & documentation**: project wiki, onboarding docs, runbooks, troubleshooting guides, decision logs.
- **How-to / SOP generation**: step-by-step procedures grounded in sources and code.
- **Prompt & instruction packs**: reusable system prompts, tool instructions, evaluation prompts, agent playbooks.
- **Study artifacts**: flashcards, quizzes, structured tables, timelines.
- **Media artifacts**: audio (TTS), images, optional video.

## MCP-first ingestion
MCP servers expand inputs safely:
- Drive, GitHub, Notion, crawlers, databases, ticketing systems, etc.
- Notebook admins can enable/disable connectors; access is permissioned and audited.

## Sharing & collaboration
First-class sharing for notebooks, sources, chats, and artifacts via share links + role-based permissions.

## Observability & cost control
Track tokens/cost/latency, context window usage, storage usage, retrieval stats, and provider errors/retries.

## Success criteria
- Swap relational DB and vector store independently.
- Add/remove providers without rewriting core logic.
- Deploy to Abacus, VPS, or Netlify+API with minimal changes.
- UI reflects capabilities; no broken actions exposed.
