# BSNIO Notebook Research (NotebookLM-style Clone)

An **open, deploy-anywhere, provider-agnostic** research notebook application inspired by NotebookLM.

This project is built around three ideas:
1) **Bring your own providers** (Gemini, Claude, OpenAI, OpenRouter, Abacus RouteLLM)
2) **Bring your own data/retrieval** (Gemini File Search, Postgres+pgvector, Abacus Vector Store)
3) **Capability-driven UX** (the app introspects config and adapts features automatically)

---

## What the app does

### Core workflow
- Create a **notebook**
- Add **sources** (files, URLs, YouTube transcripts, pasted text, code repos)
- Chat with your sources using **RAG** (retrieval-augmented generation) with **citations**
- Generate **artifacts** (“Studio outputs”) from those sources

### Artifacts (outputs)
- Reports / briefings / structured summaries
- Flashcards + quizzes + study guides
- Slides (PPTX) + downloadable exports
- Mind maps (graph/outline views)
- Audio overview (podcast-style)
- Video overview (optional)
- Codebase understanding: wiki/docs/how-to + “prompt packs” for putting knowledge to work

### Sharing
- Share a notebook (viewer/editor)
- Share a chat session
- Share specific artifacts (e.g., an audio overview, a slide deck)

### Observability
- Token usage + cost estimates per operation
- Storage usage, source tokenization metrics, context-window metrics
- Admin-level audit logs

---

## Tech stack (paved road)

- **Backend API:** Bun + TypeScript
- **HTTP framework:** lightweight TS HTTP framework (implementation guide uses a Hono-style approach)
- **Data:** Postgres (Supabase or direct), optional pgvector
- **Retrieval:** Gemini File Search OR pgvector OR Abacus Vector Store
- **Auth:** Supabase Auth OR OAuth/OIDC OR JWT-only
- **Storage:** Supabase Storage OR S3-compatible OR VPS filesystem
- **Frontend:** Next.js + Tailwind + shadcn/ui (or any client; UI is just an API consumer)

---

## Deployment options

Choose a **Deployment Profile** (documented in the reference doc):

1) **Netlify + VPS (recommended baseline)**
   - Frontend: Netlify
   - Backend: Bun API on VPS (Hostinger/Contabo/etc)

2) **Abacus-hosted (Abacus ecosystem)**
   - App deployed/hosted in Abacus
   - RouteLLM as OpenAI-compatible router
   - Retrieval via Abacus Vector Store (or external pgvector)

3) **Vercel fullstack (frontend + separate backend)**
   - Frontend on Vercel
   - Backend on VPS/Abacus (recommended) unless your environment supports Bun runtime

4) **VPS monolith**
   - Everything behind Nginx/Traefik

---

## Documentation

These docs are the authoritative source of truth:

- **Vision:** `docs/01_BSNIO-Notebook-Research_VISION_DOCUMENT.md`
- **Specification:** `docs/02_BSNIO-Notebook-Research-PROJECT_SPECIFICATION.md`
- **Implementation Guide:** `docs/03_BSNIO-Notebook-Research-IMPLEMENTATION_GUIDE.md`
- **Docs Update Plan (history / rationale):** `docs/04_BSNIO-Notebook-Research_DOCS_UPDATE_PLAN_2026_v2.md`
- **Reference (providers/capabilities/env/deploy):** `docs/05_BSNIO-Notebook-Research_CAPABILITIES_PROVIDERS_DEPLOYMENT.md`

---

## Quick start (high level)

1) Pick a deployment profile.
2) Configure env vars for:
   - auth
   - db/storage
   - at least one LLM provider
   - one retrieval backend
3) Run migrations.
4) Start API and UI.

See the Implementation Guide for the exact steps.

---

## Contributing

- Keep all new features capability-gated.
- Avoid provider lock-in.
- Keep the API stable; UI can change freely.

