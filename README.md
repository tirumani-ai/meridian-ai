# Meridian AI

Meridian AI is a FastAPI + React application that combines two GenAI patterns behind one API:

1. **Document Q&A (RAG).** Upload PDFs (policies, contracts, reports) and ask questions in natural language. Answers are grounded only in the retrieved document chunks; out-of-scope questions get an explicit "I don't know".
2. **Multi-agent purchase audit.** A purchase request is reviewed by three tool-using LangChain agents (risk, tax, internal control). A final CFO step synthesises their reports into one memo with a decision: **approved**, **conditional hold**, or **rejected**.

The same codebase runs **fully local** (Gemini API + Chroma) or **on Google Cloud** (Vertex AI embeddings, Vertex AI Vector Search, Cloud Storage, Secret Manager, Cloud Run), switched by one setting: `APP_MODE`.

The sample company, *Aldermoor Industries*, and the documents in `sample_docs/` are fictional.

**Jump to:** [Architecture](#architecture) · [Tech stack](#tech-stack) · [Design decisions](#design-decisions) · [Repository layout](#repository-layout) · [Run locally](#run-locally) · [Deploy](#deploy-to-google-cloud) · [Settings](#settings) · [Limitations](#limitations-and-next-steps) · [Troubleshooting](#troubleshooting)

---

## Architecture

```
 Browser
   │
   ▼
 Frontend (React + TypeScript, Vite)
   │  REST (src/lib/api.ts)
   ▼
 Backend (FastAPI)
   │
   ├── Document Q&A (RAG)
   │     1. PDF → chunks (recursive splitter with overlap)
   │     2. chunks → embeddings ──────────► Gemini / Vertex AI embeddings
   │     3. store ────────────────────────► Chroma (local) | Vertex AI Vector Search (cloud)
   │     4. question → top-k chunks ◄──────┘
   │     5. grounded prompt ──────────────► Gemini
   │
   └── Purchase audit (agents)
         Risk agent    ─ tools ┐
         Tax agent     ─ tools ├─► Gemini ─► CFO synthesis ─► memo + decision
         Control agent ─ tools ┘
         (LangChain v1 middleware: PII redaction, call limits, retries, tool-call logging)
```

**RAG request flow:** the question is embedded, the vector store returns the three nearest chunks, and Gemini answers using only that context.

**Audit flow:** each specialist agent decides which tools to call (e.g. a live FX-rate lookup), writes a short report, and the CFO step reads all three reports and produces the final memo.

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React, TypeScript, Vite |
| API | FastAPI, Pydantic (request/response schemas), pydantic-settings |
| LLM orchestration | LangChain v1 (agents, tools, middleware) |
| Models | Gemini (chat), `gemini-embedding-001` (local) / `text-embedding-005` (cloud) |
| Vector store | Chroma (local), Vertex AI Vector Search (cloud) |
| Storage & secrets | Cloud Storage, Secret Manager |
| Runtime | Docker, Cloud Run |
| CI/CD | GitHub Actions (`gcloud`-based provisioning and deploy) |
| Observability | structlog (JSON logs, shipped to Cloud Storage on shutdown) |
| Testing | pytest, httpx, fake LLMs (no network or accounts required) |

---

## Design decisions

- **One switch for local vs cloud.** `APP_MODE` selects the embedding model, vector store and file storage; nothing else in the code branches on environment. Local mode needs only a Gemini API key.
- **Layered packages.** `api → rag, agent → config, logger`. The web layer knows nothing about LLMs, and the RAG/agent code knows nothing about HTTP, so each part can be tested on its own.
- **Lazy external clients.** Nothing external is created at import or start-up, so the server boots and the test suite runs without cloud credentials.
- **Agent guardrails as middleware.** Model-call limits, tool-call limits, retries, PII redaction and tool-call logging wrap every agent step instead of being scattered through prompts.
- **Structured logging.** Every event is one JSON line, which makes failures searchable locally and in Cloud Storage.
- **Infrastructure via GitHub Actions + `gcloud`.** No Terraform; the workflow provisions the bucket, vector index, endpoint and Cloud Run service idempotently.

More detail on the trade-offs is in [docs/17-why-built-this-way.md](docs/17-why-built-this-way.md) and [docs/20-local-cloud-and-middleware.md](docs/20-local-cloud-and-middleware.md).

---

## Repository layout

```
backend/
├── api/                      FastAPI app
│   ├── main.py                 App factory, middleware, routers
│   ├── endpoints.py            Routes (e.g. POST /api/rag/ask, POST /api/agent/audit)
│   └── schemas.py              Pydantic request/response models
├── rag/                      Document Q&A
│   ├── llm.py                  Gemini chat model
│   ├── embeddings.py           Embedding model (per APP_MODE)
│   ├── vector_store.py         Chroma / Vertex AI Vector Search
│   ├── data_ingestion.py       PDF → chunks → vector store
│   └── retrieval.py            Question → retrieve → grounded answer
├── agent/                    Purchase audit
│   ├── tools.py                Agent tools
│   ├── prompts.py              Specialist and CFO prompts
│   └── agents.py               Agent construction and orchestration
├── config/settings.py        Typed settings from environment
├── logger/custom_logger.py   structlog JSON logging
├── exceptions/               Application errors
└── tests/                    Offline test suite

frontend/                     React + TypeScript UI (src/lib/api.ts is the only API client)
sample_docs/                  Four fictional PDFs for trying the app
docs/                         Architecture and component notes, runnable examples in docs/examples/
.github/workflows/deploy.yml  Provisioning + deploy pipeline
Dockerfile                    Production image
DEPLOY.md                     Google Cloud deployment guide
```

---

## Run locally

### Prerequisites

| For | You need |
|---|---|
| Backend | Python 3.12, Git |
| Frontend | Node.js 20+ |
| Local mode (audit and Q&A) | A Gemini API key from https://aistudio.google.com/apikey |
| Cloud mode | A Google Cloud project (see [DEPLOY.md](DEPLOY.md)) |

### Local vs cloud mode

| | `APP_MODE=local` (default) | `APP_MODE=cloud` |
|---|---|---|
| Chat model | Gemini API (API key) | Gemini API (key from Secret Manager) |
| Embeddings | `gemini-embedding-001` via Gemini API | `text-embedding-005` via Vertex AI |
| Vector database | Chroma on disk (`.meridian/chroma`) | Vertex AI Vector Search |
| Uploaded PDFs | `.meridian/uploads/` | Cloud Storage bucket |
| Logs | `logs/` | `logs/` + copied to the bucket on shutdown |

`GET /api/status` (and the System Status tab) reports the active mode.

### Backend

```bash
git clone https://github.com/tirumani-ai/meridian-ai.git
cd meridian-ai

python3.12 -m venv .venv
source .venv/bin/activate            # Windows PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt

cp .env.example .env                 # then set GOOGLE_API_KEY=... (keep APP_MODE=local)

uvicorn api.main:app --app-dir backend --reload --port 8080
```

Open http://localhost:8080/docs for the interactive API. A sample audit request for `POST /api/agent/audit`:

```json
{"request_text": "Purchase of 200 PLC controllers from Takumi Controls Europe B.V. (Netherlands). Total 480,000 EUR. Origin JP, destination DE. FX rate quoted: 1 EUR = 163.5 JPY."}
```

The response contains the risk, tax and control reports plus the CFO memo. An audit makes several model calls and can take a minute.

Run commands from the repository root; settings are read from `.env` in the working directory.

### Frontend

```bash
cd frontend
npm install
npm run dev                          # http://localhost:3000
```

### Try document Q&A

1. Upload the PDFs in `sample_docs/` (Index Documents tab, or `POST /api/rag/upload`).
2. Ask: *"What is Aldermoor Industries total revenue in FY2024?"* → about **EUR 2.84 billion**. Other checks: payment terms for servo motor suppliers (Net-45), customs duty on PLC controllers from Japan (2.2 %), depreciation of filling line equipment (10 years).
3. Ask something not covered by the documents; the answer should say it does not know.

### Tests

```bash
pytest backend/tests
```

17 tests, no accounts or network needed. They include an end-to-end local RAG run (PDF → Chroma → answer) and an agent run through the middleware, using fake models.

### Docker

```bash
docker build -t meridian-ai .
docker run -p 8080:8080 --env-file .env -v "$(pwd)/.meridian:/app/.meridian" meridian-ai
```

The volume keeps the local vector store and uploads across container restarts.

---

## Deploy to Google Cloud

The GitHub Actions workflow in `.github/workflows/deploy.yml` provisions the Cloud Storage bucket, Vertex AI Vector Search index and endpoint (and enables the required APIs, including Secret Manager), then builds the image and deploys it to Cloud Run. See [DEPLOY.md](DEPLOY.md) for setup, required secrets, costs and teardown.

> Vertex AI Vector Search endpoints are billed per hour while deployed, even when idle. Tear them down when not in use.

---

## Settings

Settings live in `.env` (copy `.env.example`). `.env` is git-ignored.

| Setting | Meaning | Needed for |
|---|---|---|
| `APP_MODE` | `local` (default) or `cloud` | Always |
| `GOOGLE_API_KEY` | Gemini API key | Both modes |
| `VERTEX_LLM_MODEL_NAME` | Chat model, default `gemini-3.8-flash` | Optional |
| `LLM_TEMPERATURE` | Sampling temperature (`0` = most deterministic) | Optional |
| `LOCAL_EMBEDDING_MODEL_NAME` | Default `gemini-embedding-001` | Optional, local |
| `LOCAL_DATA_DIR` | Chroma and uploads location, default `.meridian/` | Optional, local |
| `AGENT_MAX_MODEL_CALLS` / `AGENT_MAX_TOOL_CALLS` / `AGENT_MAX_RETRIES` | Middleware limits, defaults 8 / 6 / 2 | Optional |
| `GCP_PROJECT_ID` | Google Cloud project | Cloud |
| `GCP_REGION` | e.g. `us-central1` | Cloud |
| `GCS_BUCKET_NAME` | Cloud Storage bucket | Cloud |
| `GCS_PREFIX` | Upload prefix, `uploads/` | Cloud |
| `VECTOR_SEARCH_INDEX_ID` | Vector Search index ID | Cloud |
| `VECTOR_SEARCH_INDEX_ENDPOINT_ID` | Index endpoint ID | Cloud |
| `GCP_SERVICE_ACCOUNT_PATH` | Service account key path | Cloud mode from a workstation |
| `VERTEX_EMBEDDING_MODEL_NAME` | Default `text-embedding-005` | Optional, cloud |

If you get "model not found", set `VERTEX_LLM_MODEL_NAME` to a current model from https://ai.google.dev/gemini-api/docs/models.

---

## Limitations and next steps

- **Audit tools are mostly LLM-backed.** Only the FX-rate tool calls a live data source; the sanctions, credit and tax checks use the model's general knowledge. A production version would call a sanctions-screening API, a credit bureau and a tax engine. The audit output is decision support, not a compliance determination.
- **No authentication.** The Cloud Run service is deployed with `--allow-unauthenticated`; add IAP or an auth layer before using real documents.
- **Logs contain prompts and answers.** Treat log files and the log bucket as sensitive.
- **Retrieval is plain top-k similarity.** Hybrid search, re-ranking and an evaluation set would be the next improvements.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `ModuleNotFoundError: No module named 'api'` | Run from the repo root with `--app-dir backend` |
| Settings appear empty | Run from the repo root (where `.env` is) |
| Other `ModuleNotFoundError` | Activate `.venv` and `pip install -r requirements.txt` |
| `503` "APP_MODE=cloud but these settings are empty" | Fill in the listed settings or set `APP_MODE=local` |
| `503` "GOOGLE_API_KEY is empty" | Add the key to `.env` and restart |
| Stale answers after changing documents | Local mode: delete `.meridian/` and re-upload |
| `403` from Google Cloud | Check the active account and IAM roles; see [DEPLOY.md](DEPLOY.md) |
| "model not found" | Update `VERTEX_LLM_MODEL_NAME` |
| Port already in use | Use `--port 8081` |
| CORS error in the browser | The backend is not running on port 8080 |

The backend logs one JSON line per event; the failing step is usually in the last few lines.

More: [docs/18-troubleshooting.md](docs/18-troubleshooting.md).

---

## Acknowledgements

Built while working through the Meridian AI course by Mayank ([reference implementation](https://github.com/mayank953/meridian-ai-learner)). Project scaffolding, sample documents and parts of the configuration follow that reference.
