# ITSM AI Copilot (RAG + Actions)

A lightweight **helpdesk copilot** that answers with citations, drafts ServiceNow‑formatted replies/KB articles, and triggers **mocked actions** (e.g., `reset_password`, `get_device_info`) behind a simple RBAC + HIL approval flow.

> **MVP focus:** Local folder ingestion only, provider‑agnostic LLM via **OpenRouter** *or* **Local‑only** mode. Actions are mocked for demo safety.

---

## Why

Helpdesk teams burn time searching KBs, formatting replies, and performing repetitive tasks. This copilot reduces **time‑to‑first‑response (TTR)** and increases **deflection** without changing your existing ITSM.

**Demo KPI targets (simulated):**

* TTR ↓ **30–40%**
* Deflection ↑ **20–30%**
* **10+** KB drafts/week

---

## Features (MVP)

* **RAG Q\&A with citations** from your KB/runbooks (Markdown/HTML/PDF)
* **ServiceNow reply format**: outputs **Public Comment** + **Work Notes** blocks
* **Drafts**: ticket reply + KB article (Markdown) with sources
* **Mocked actions** with **RBAC** + **HIL approval** (`reset_password`, `get_device_info`)
* **Model‑agnostic**: any OpenRouter model *or* local LLM (vLLM/Ollama)
* **Observability**: structured JSON logs (latency, tokens, cost, citations, actions)
* **Eval harness**: 30‑question offline evaluation
* **Dockerized**: `docker compose up` to run

---

## Architecture

```mermaid
flowchart LR
  UI[Web UI / Swagger] --> API(FastAPI /api)
  subgraph Backend
    API --> RET[(Retriever pgvector/FAISS)]
    API --> LLM{Model Provider\n(OpenRouter or Local)}
    API --> ACT[Actions Mock\nRBAC + HIL]
    API --> LOG[Structured Logs]
    W(Worker: Celery) -->|Ingest/Eval| DB[(Postgres + pgvector)]
    RET --> DB
  end
  FS[[Local KB Folder]] -->|Ingest| W
```

---

## Quickstart

### 1) Prereqs

* Docker & Docker Compose
* A folder of KB files (Markdown/HTML/PDF) under `./data/kb`

### 2) Configure environment

Create `.env` in the repo root:

```env
# Provider: openrouter or local
PROVIDER=openrouter
OPENROUTER_API_KEY=your_key_here        # required if PROVIDER=openrouter
MODEL_ID=anthropic/claude-3.5-sonnet    # any model supported by OpenRouter
EMBED_MODEL_ID=openai/text-embedding-3-large
LOCAL_ONLY=false                        # true → use local LLM + local embeddings

# Local LLM (if LOCAL_ONLY=true)
LOCAL_LLM_ENDPOINT=http://localhost:8001/v1  # vLLM/Ollama compatible
LOCAL_EMBED_MODEL=bge-large-en

# Database & auth
DB_URL=postgresql+psycopg2://postgres:example@db:5432/itsm
JWT_PUBLIC_KEY=dev                       # demo-only placeholder

# RAG
TOP_K=4
MAX_TOKENS_OUT=900

# Budgets
DAILY_USD_BUDGET=5.00
```

### 3) Start the stack

```bash
docker compose up --build
```

* API: [http://localhost:8000/docs](http://localhost:8000/docs) (Swagger)
* Postgres: exposed on 5432 (dev only)

### 4) Ingest your KB

Place files under `./data/kb` (mounted at `/data/kb` in the API container), then:

```bash
curl -X POST http://localhost:8000/api/admin/ingest \
  -H 'Content-Type: application/json' \
  -d '{"path":"/data/kb"}'
```

### 5) Ask a question (ServiceNow format)

```bash
curl -X POST http://localhost:8000/api/chat \
  -H 'Content-Type: application/json' \
  -d '{
    "message": "User cannot access VPN on Windows 11; what are the steps?",
    "mode": "steps",
    "reply_format": "servicenow"
  }'
```

**Response (truncated):**

```json
{
  "answer_markdown": "**Public Comment**\nPlease follow these steps...\n\n**Work Notes**\nRoot cause likely...\nSources: [1] [2]",
  "sources": [
    {"id":1,"title":"VPN Troubleshooting","path":"/kb/vpn/troubleshooting.md"}
  ]
}
```

### 6) Try a mocked action (HIL approval)

```bash
# Propose reset
curl -X POST http://localhost:8000/api/actions/reset_password \
  -H 'Content-Type: application/json' \
  -d '{"username":"jdoe"}'

# Approve as approver/admin
curl -X POST http://localhost:8000/api/actions/approve \
  -H 'Content-Type: application/json' \
  -d '{"action_id":"<from_previous_step>","approve":true}'
```

**Demo accounts (mocked):**

* Agent: `sam.agent@demo.local`
* Approver: `alex.approver@demo.local`
* Admin: `riley.admin@demo.local`

> **Note:** In the demo, JWT/RBAC is stubbed for local use. Do **not** deploy as‑is to production.

---

## Project Structure

```
.
├─ api/            # FastAPI app (chat, drafts, actions, admin)
├─ worker/         # Celery tasks (ingest, eval, rollups)
├─ retrieval/      # loaders, chunkers, embeddings, index utils
├─ frontend/       # optional minimal UI (coming soon)
├─ infra/          # docker-compose, DB migrations, OpenAPI build
├─ docs/           # PRD, style guides, eval set (30 Qs)
├─ scripts/        # ingest/eval helpers
├─ data/kb/        # your source KB files (mounted in containers)
└─ README.md
```

---

## ServiceNow Reply Style

The copilot outputs two blocks:

```markdown
**Public Comment**
Plain‑English steps for the requester. Keep it concise; include validation.

**Work Notes**
Technical context for agents: diagnostics, links, and any action payloads.

**Sources**
[1] Title (path)
[2] Title (path)
```

---

## Evaluation (30‑Q Harness)

* Dataset lives under `./docs/eval/questions.jsonl` (example provided).
* Run the eval job (inside the worker container):

```bash
docker compose exec worker python -m eval.run \
  --dataset /app/docs/eval/questions.jsonl \
  --api http://api:8000
```

* Outputs CSV/JSON scores for **Groundedness, Exactness, Usefulness, Citation Correctness**.

---

## Observability

* Structured JSON logs with `trace_id`, latency, token usage, USD cost, citations, actions.
* Tail logs:

```bash
docker compose logs -f api worker
```

---

## Roadmap

* Source connectors (Confluence/SharePoint/Drive)
* Real action plugins (password reset, device mgmt) with secrets vault + approval queues
* Cross‑encoder re‑ranking & summarization cache
* SSO/OIDC, multi‑tenant RBAC, audit exports

---

## Security & License

* **Security:** Actions are mocked; JWT/RBAC is demo‑only. Review before any production use.
* **License:** MIT (see `LICENSE`).

---

## Contributing

Issues and PRs welcome. Please include:

* Clear repro steps, logs, and example inputs/outputs.
* For features, attach a short design note and sample API payloads.

---

## Acknowledgments

Built with FastAPI, Postgres + pgvector, Celery, Docker, and a provider‑agnostic LLM interface (OpenRouter or Local‑only mode).
