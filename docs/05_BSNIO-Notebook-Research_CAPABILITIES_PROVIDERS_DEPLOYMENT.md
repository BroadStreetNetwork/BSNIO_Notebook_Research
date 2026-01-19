# BSNIO Notebook Research — Reference (V2.1)
Capabilities • Providers • Data Options • Env Vars • Profiles • PR Checklist

## 1) Provider matrices

### 1.1 LLM providers
| Provider | Key | API style | Notes |
|---|---|---|---|
| Gemini | GEMINI_API_KEY | native | pairs well with Gemini File Store + Imagen |
| Anthropic | ANTHROPIC_API_KEY | native | strong coding/reasoning |
| OpenAI | OPENAI_API_KEY | openai | baseline |
| OpenRouter | OPENROUTER_API_KEY | openai-compatible | one endpoint, many models |
| RouteLLM | ROUTELLM_API_KEY | openai-compatible | policy-based routing |

### 1.2 Artifact providers
| Capability | Provider options | Notes |
|---|---|---|
| TTS | Gemini, OpenAI, others | capability-gated |
| Image | Gemini Imagen, OpenAI Images, OpenRouter models | first-class artifact |
| Video | AtlasCloud + others + Abacus tools | optional |

## 2) Data plane options (independent)

### 2.1 Relational DB
| Option | Type | Typical use |
|---|---|---|
| Postgres | server | multi-user OLTP |
| Supabase Postgres | managed | fast managed path |
| MySQL | server | ubiquitous hosting |
| SQLite | embedded | lightweight/dev/VPS |
| Turso | hosted SQLite | SQLite portability + hosted |
| DuckDB | embedded analytics | analysis-heavy/offline |
| Abacus relational | managed | optional adapter |

### 2.2 Object storage
| Option | Type | Notes |
|---|---|---|
| Supabase Storage | managed | convenient with Supabase |
| S3-compatible | managed/self | S3/R2/etc. |
| Abacus Storage | managed | adapter |
| Local FS | local | dev/VPS |

### 2.3 Vector/RAG
| Provider | Type | Works with relational DB | Notes |
|---|---|---|---|
| Gemini File Store | managed | any | managed semantic retrieval + metadata filters |
| pgvector | Postgres extension | Postgres/Supabase | integrated vectors |
| sqlite-vector | SQLite extension | SQLite/Turso | embedded vectors |
| Qdrant | service | any | dedicated vector DB |
| LanceDB | embedded | any | local-first |
| Abacus Vector | managed | any | adapter |
| vectorwrap | wrapper | any | unify multiple backends |

### 2.4 Constraints
- VECTOR_PROVIDER=pgvector requires RELATIONAL_PROVIDER=postgres|supabase_postgres
- VECTOR_PROVIDER=sqlite_vector requires RELATIONAL_PROVIDER=sqlite|turso

## 3) Capabilities (example)
```json
{
  "deploy": { "profile": "vps-docker" },
  "data": {
    "relational": { "provider": "mysql", "ready": true },
    "storage": { "provider": "s3", "ready": true },
    "vector": { "provider": "gemini_file_search", "ready": true, "constraints": [] }
  },
  "providers": {
    "llm": { "enabled": ["gemini","anthropic","openai","openrouter","routellm"], "default": "routellm" },
    "tts": { "enabled": ["gemini","openai"] },
    "image": { "enabled": ["gemini_imagen","openai_images"] },
    "video": { "enabled": [] }
  },
  "features": {
    "sharing": true,
    "artifacts": true,
    "mcp": true,
    "multimodel_chat": true,
    "codebase_tools": true
  }
}
```

## 4) Environment variables (reference)

### 4.1 Deploy
DEPLOY_PROFILE=abacus-hosted|vps-docker|netlify-web|vercel

### 4.2 Relational
RELATIONAL_PROVIDER=postgres|supabase_postgres|mysql|sqlite|turso|duckdb|abacus_relational
RELATIONAL_DSN=...

### 4.3 Storage
STORAGE_PROVIDER=supabase_storage|s3|abacus_storage|local_fs
STORAGE_BUCKET=...
STORAGE_ENDPOINT=... (S3/R2)
STORAGE_ACCESS_KEY=...
STORAGE_SECRET_KEY=...

### 4.4 Vector
VECTOR_PROVIDER=gemini_file_search|pgvector|sqlite_vector|qdrant|lancedb|abacus_vector|vectorwrap
VECTOR_ENDPOINT=... (qdrant/abacus/vectorwrap)
VECTOR_API_KEY=... (if needed)
VECTOR_COLLECTION_PREFIX=...

### 4.5 Auth
AUTH_PROVIDER=supabase_auth|oidc|jwt|dev
OIDC_ISSUER=...
OIDC_CLIENT_ID=...
OIDC_CLIENT_SECRET=...
JWT_PUBLIC_KEY=...

### 4.6 LLM keys
GEMINI_API_KEY=...
ANTHROPIC_API_KEY=...
OPENAI_API_KEY=...
OPENROUTER_API_KEY=...
OPENROUTER_BASE_URL=...
ROUTELLM_API_KEY=...
ROUTELLM_BASE_URL=...

## 5) Profile checklists

### VPS Docker
- [ ] relational connected
- [ ] storage write test passes
- [ ] vector ready + constraints satisfied
- [ ] worker running
- [ ] capabilities endpoint shows ready components

### Netlify + external API
- [ ] web deployed
- [ ] API/worker reachable
- [ ] CORS configured
- [ ] OAuth callbacks configured

### Abacus-hosted
- [ ] API + worker deployed
- [ ] optional Abacus DB/vector/storage configured
- [ ] optional RouteLLM routing configured
- [ ] capabilities correct

## 6) PR checklist
- [ ] docs 01–05 consistent
- [ ] README updated
- [ ] independence rule stated clearly
- [ ] constraints explicit
- [ ] capabilities + env vars reflect all options
