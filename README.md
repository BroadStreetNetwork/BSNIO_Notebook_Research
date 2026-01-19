# BSNIO Notebook Research

A multi-provider, deploy-anywhere NotebookLM-style research workspace that turns **sources → grounded understanding → reusable outputs**.

## What it does
- Ingest sources: file uploads, URLs, and MCP connectors (Drive, GitHub, Notion, etc.)
- Retrieve evidence via a configurable RAG backend (managed or self-hosted) and answer with citations
- Generate reusable outputs (“put learning to work”):
  - codebase understanding + repo wikis
  - documentation/how-to/SOPs
  - prompt & instruction packs
  - tables, slide outlines, flashcards/quizzes
  - audio (TTS), images, optional video

## V2 tech direction
- Web: Next.js
- API + workers: Bun (TypeScript)
- Architecture: adapters for Providers, Data Plane, Auth, and Deployment Profiles

## Pluggable options

### LLM providers (BYO keys)
- Google Gemini
- Anthropic Claude
- OpenAI
- OpenRouter (OpenAI-compatible)
- Abacus RouteLLM (OpenAI-compatible router)

### Data Plane (three independent decisions)

**Relational DB (metadata/transactions)**
- Postgres / Supabase Postgres
- MySQL
- SQLite (embedded)
- Turso (hosted SQLite)
- DuckDB (embedded analytics)
- Abacus transactional DB (adapter, if/when used)

**Vector Store / RAG Index (retrieval)**
- Gemini File Search ("Gemini File Store") — managed semantic retrieval + metadata filters
- Postgres + pgvector (requires Postgres)
- SQLite/Turso + sqlite-vector (requires SQLite/Turso)
- Qdrant
- LanceDB
- Abacus Vector Store (adapter)
- vectorwrap (optional wrapper/unifier across vector backends)

**Object Storage (files/artifacts)**
- Supabase Storage
- S3-compatible (AWS S3, Cloudflare R2, Wasabi, etc.)
- Abacus Storage (adapter)
- Local filesystem (VPS/dev)

### Auth
- Supabase Auth
- OAuth/OIDC (Google/GitHub)
- JWT-only for private deployments behind SSO
- Dev profile (no-auth) for local/dev

## Deploy
Deployments are defined by a **profile**:
- Abacus-hosted
- VPS Docker
- Netlify web + external API/worker
- (Optional) Vercel

## Docs
See `/docs/`:
- 01 Vision
- 02 Project Specification
- 03 Implementation Guide
- 04 Docs Update Plan
- 05 Reference (capabilities, providers, env vars, profile checklists)
- Master Doc (all docs combined for Google Docs conversion)
