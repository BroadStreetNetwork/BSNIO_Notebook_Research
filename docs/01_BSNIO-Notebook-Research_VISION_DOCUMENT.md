# BSNIO Notebook Research
## Vision Document (V2)

> **Version:** 2.0 (Docs Refresh)
>
> This project is an **open, deploy-anywhere, provider-agnostic** NotebookLM-style research workspace. The product is designed around **capabilities introspection**, **abstraction layers**, and **portable deployment profiles**, so it can run on Abacus-hosted deployments, Netlify+VPS, Vercel, or a private server—without rewriting the app.

---

## 1) Why this exists

NotebookLM is a great “thinking partner,” but it’s a closed product and a walled garden: limited provider choice, limited deployment choice, limited data/control choice.

BSNIO Notebook Research aims to be:
- **Open and flexible**: bring your own keys (Gemini, Claude, OpenAI, OpenRouter, Abacus RouteLLM), choose your retrieval backend, and deploy in multiple environments.
- **Truth-forward**: citations, source provenance, auditability, cost visibility, and strong UX cues that distinguish “in sources” vs “model inference.”
- **Practical**: turn knowledge into action—wikis, docs, how-to guides, prompts, codebase documentation, and artifacts teams can share.

---

## 2) Product principles (non-negotiables)

### 2.1 Paved roads + escape hatches
We provide a default “happy path” (Supabase + Gemini + Vercel/Netlify) but keep modular alternatives:
- Data: Supabase Postgres / direct Postgres / (optional) Abacus app DB
- Retrieval: Gemini File Search / Postgres+pgvector / Abacus Vector Store
- Storage: Supabase Storage / S3-compatible / local filesystem (VPS)
- Auth: Supabase / OAuth/OIDC / JWT-only

### 2.2 Capability-driven UX
The app is self-aware.
- A `/capabilities` endpoint drives UI availability.
- The same frontend can run against different backends and show only what’s configured.

### 2.3 Provider-agnostic by design
No “Gemini-only.” No “OpenAI-only.” The app treats providers as adapters.

### 2.4 Observability is a feature
Users should see:
- token usage, provider/model used, cost estimate/actuals
- storage usage
- context window info (what’s being fed, what got retrieved)
- per-notebook and per-user usage history

### 2.5 Sharing is first-class
Share:
- notebooks (restricted / link / invite)
- notebook sources (controlled)
- chats (sessions or specific messages)
- artifacts (audio/video/slides/mind maps/reports/flashcards)

---

## 3) What we are building

### 3.1 Core loop
1) Create a notebook
2) Add sources (files, URLs, YouTube, pasted text, optional MCP connectors)
3) Ask questions with citations and source-grounding
4) Generate artifacts (Studio outputs)
5) Share results with a team or public link

### 3.2 Feature pillars

**A) Research chat with citations**
- conversational Q&A over sources
- citations with source pointers
- “strict source mode” vs “allow inference mode”

**B) Studio (multi-output)**
- Audio Overviews (multi-format)
- Video Overviews (provider-dependent)
- Mind Maps (topic graph of sources)
- Reports/Briefings (executive summary, critique, Q&A, etc.)
- Flashcards + quizzes
- Slides (PPTX export)

**C) Put learning to work (action outputs)**
- Codebase understanding (repo ingestion + architecture explainer)
- Wiki/documentation generation
- How-to guides and SOPs
- Prompt packs / reusable instruction templates

**D) Source discovery**
- optional “discover sources” mode (web search + suggested sources)
- imports become notebook sources (with provenance)

---

## 4) Architecture shape (conceptual)

### 4.1 Abstraction layers (V2)
- **Provider Abstraction Layer**: LLM, embeddings, rerank, image, TTS, video, web-search.
- **RAG Index Abstraction**: retrieval backend is swappable.
- **Data Abstraction Layer**: DB + Storage (Supabase / Postgres / S3).
- **Auth Abstraction Layer**: identity providers + JWT/API keys.
- **Deployment Profile Layer**: “Abacus-hosted”, “Netlify+VPS”, “Vercel Fullstack”, “VPS Monolith”.

### 4.2 Bun-first backend
V2 pivots backend to **Bun + TypeScript** for:
- single-language stack (TS across backend/frontend)
- simpler refactoring and adapter development
- easier packaging for VPS or container deployments

(Local desktop packaging is optional; not a required deliverable in the docs refresh.)

---

## 5) Non-goals (for this docs refresh)

- Perfect 1:1 replication of every NotebookLM detail
- Heavy multi-tenant enterprise RBAC on day one
- A single “one true” deployment platform

---

## 6) Success criteria

### 6.1 Technical success
- Swap providers without rewriting product code
- Swap retrieval backends without rewriting chat logic
- Deploy to 2+ platforms with minimal changes

### 6.2 Product success
- A notebook can be created and shared
- Sources can be imported from multiple formats
- Chat produces grounded answers with citations
- At least 3 artifact types are working end-to-end

---

## 7) References inside the docs set

- Implementation sequence + refactor plan: `04_BSNIO-Notebook-Research_DOCS_UPDATE_PLAN_2026_v2.md`
- Canonical configuration matrices: `05_BSNIO-Notebook-Research_CAPABILITIES_PROVIDERS_DEPLOYMENT.md`
