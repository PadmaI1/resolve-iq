<p align="center">
  <img src="assets/logo.svg" alt="ResolveIQ" width="420">
</p>

<p align="center">
  A production-ready RAG-powered customer support assistant built with FastAPI, FAISS, and Azure OpenAI.<br>
  Semantic retrieval, query rewriting, grounded responses, session management, and hallucination controls.
</p>

<p align="center">
  <img alt="Python" src="https://img.shields.io/badge/python-3.11%2B-2563eb">
  <img alt="FastAPI" src="https://img.shields.io/badge/FastAPI-009688">
  <img alt="FAISS" src="https://img.shields.io/badge/vector%20store-FAISS-6d28d9">
  <img alt="Azure OpenAI" src="https://img.shields.io/badge/LLM-Azure%20OpenAI-0078d4">
  <img alt="Tests" src="https://img.shields.io/badge/tests-offline%20%26%20deterministic-16a34a">
</p>

---

## Overview

ResolveIQ answers customer questions strictly from a curated knowledge base. If the
knowledge base cannot support an answer, it says so instead of guessing. The
bundled knowledge base covers Amenify's resident services, but the pipeline is
content-agnostic: swap the JSON files in `Backend/data/` to support any product.

## Features

- **Semantic retrieval** — word-based chunks embedded and searched with a FAISS `IndexFlatIP` index (cosine similarity).
- **Query rewriting** — follow-ups such as "who founded it?" are rewritten into standalone queries using session history.
- **Hallucination controls** — a 0.75 similarity gate (no LLM call below it), a strict context-only system prompt, and `temperature=0`.
- **Grounded, cited answers** — structured JSON replies with source citations and a `found_in_kb` flag.
- **Session management** — in-memory history, 10 messages per session, 5,000 sessions with LRU eviction, and a clear-conversation endpoint.
- **Flexible ingestion** — manual JSON, then scraped cache, then live scrape.
- **Production hardening** — fail-fast config validation, configurable CORS, optional per-IP rate limiting, a `/health` check that reports `degraded`.
- **Observability** — one structured JSON log record per chat request (raw query text is off by default).
- **Chat UI** — static vanilla-JS frontend served by the app.
- **Tested** — offline, deterministic pytest suite plus an opt-in live suite, with CI on every push.

## Architecture

```
User message
    ↓
Query Rewriter        — handles follow-ups like "who founded it?"
    ↓
FAISS Vector Search   — semantic search over knowledge-base chunks
    ↓
Similarity Gate       — score < 0.75 → "I don't know" (no LLM call)
    ↓
Azure OpenAI GPT      — answers from retrieved context only
    ↓
Response              — answer + source citations + session_id
```

| Layer | Tech |
|---|---|
| Backend | FastAPI (Python 3.11+) |
| LLM | Azure OpenAI GPT-4 |
| Embeddings | Azure `text-embedding-ada-002` |
| Vector store | FAISS `IndexFlatIP` |
| Sessions | In-memory (`OrderedDict`, LRU) |
| Frontend | HTML / CSS / vanilla JS |
| Hosting | Azure App Service (Linux) |

## Project structure

```
ResolveIQ/
├── Backend/
│   ├── main.py            # FastAPI app, routes, lifespan, CORS, rate limiter, static mount
│   ├── config.py          # Env loading, fail-fast validation, all tunables
│   ├── ingestion.py       # KB loader, scraper, word-based chunker
│   ├── vector_store.py    # Embedding pipeline + FAISS index builder (CLI)
│   ├── retriever.py       # Query embedding + top-k search with similarity gate
│   ├── chat.py            # RAG orchestration + structured request log
│   ├── session.py         # In-memory chat history
│   ├── logging_config.py  # JSON logging
│   ├── models.py          # Pydantic models
│   ├── tests/             # pytest suite (offline + opt-in live)
│   └── data/              # knowledge base + FAISS index
├── frontend/              # chat UI served at /
├── assets/                # logo files
├── docs/superpowers/      # design spec + implementation plan
├── .github/workflows/     # Azure deploy + offline test workflow
├── pyproject.toml         # local dev/test deps (uv)
├── Dockerfile
└── requirements.txt       # → Backend/requirements.txt (Azure deploy lockfile)
```

## Getting started

**Prerequisites:** Python 3.11+ and an Azure OpenAI resource with a GPT deployment and a `text-embedding-ada-002` deployment.

```bash
git clone https://github.com/PadmaI1/resolve-iq.git
cd resolve-iq/Backend
pip install -r requirements.txt
cp .env.example .env     # then fill in your Azure values
python vector_store.py   # build the FAISS index
python -m uvicorn main:app --reload
```

Open http://localhost:8000 for the chat UI, `/docs` for the API docs, `/health` for status.

### Configuration

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `AZURE_OPENAI_API_KEY` | yes | — | Azure OpenAI key |
| `AZURE_OPENAI_ENDPOINT` | yes | — | `https://<resource>.openai.azure.com/` |
| `AZURE_OPENAI_DEPLOYMENT_NAME` | yes | — | GPT deployment name |
| `AZURE_OPENAI_API_VERSION` | yes | — | e.g. `2024-02-01` |
| `AZURE_EMBEDDING_MODEL` | yes | — | e.g. `text-embedding-ada-002` |
| `RATE_LIMIT` | no | `20/minute` | Per-IP `/chat` limit; empty disables (needs `slowapi`) |
| `ALLOWED_ORIGINS` | no | `*` | CORS origins (`*` or comma-separated) |
| `LOG_LEVEL` | no | `INFO` | `DEBUG` \| `INFO` \| `WARNING` \| `ERROR` |
| `LOG_QUERY_TEXT` | no | `false` | Log the raw user message (PII — off by default) |

### Docker

```bash
docker build -t resolveiq .
docker run --rm -p 8000:8000 --env-file Backend/.env resolveiq
```

### Tests

The offline suite needs no API key or network: Azure calls are mocked and FAISS is built in memory.

```bash
uv sync --group dev
uv run pytest                 # offline
uv run pytest -m live         # also hits real Azure (needs AZURE_OPENAI_* vars)
uv run pytest --cov=Backend   # coverage
```

`Backend/requirements.txt` is the source of truth for the Azure deploy; `pyproject.toml` and `uv.lock` are for local development. If a runtime dependency changes, update both (`uv export --no-hashes --no-dev`).

## API

### `POST /chat`

```json
// Request
{ "message": "What services does Amenify offer?", "session_id": null }

// Response
{
  "answer": "Amenify offers house cleaning, handyman services, dog walking...",
  "session_id": "abc-123",
  "sources": ["Amenify Services Overview"],
  "found_in_kb": true
}
```

Pass `session_id` back on later requests to keep conversation history. Over the rate limit the endpoint returns `429`.

### `GET /health`

```json
{ "status": "ok", "index_loaded": true, "total_chunks": 19 }
```

`status` is `"degraded"` when the index failed to load.

### `DELETE /session/{session_id}`

Clears a session's history. Returns `204`, and an unknown id also returns `204`.

## Reasoning & Design

### Ingestion

1. Load the hand-curated JSON (primary), else the scraped cache, else scrape live with `requests` + `BeautifulSoup`.
2. Strip `<script>`, `<nav>`, `<footer>` and lines shorter than 30 characters.
3. Split into ~500-word chunks with 50-word overlap; drop a trailing chunk of 25 words or fewer.
4. Tag each chunk `{text, source, url, chunk_id}`, embed it, and normalise vectors so inner product equals cosine similarity.

### Reducing hallucinations

1. **Similarity gate.** `MIN_SIMILARITY_SCORE = 0.75`. On this knowledge base, unrelated questions top out near 0.73 and relevant ones score 0.80–0.94, so 0.75 sits in the gap. Below it, the LLM is never called.
2. **Strict system prompt.** Answer only from the supplied context, never invent prices, phone numbers, dates or URLs, and return the fallback message when the context lacks the answer.
3. **`temperature=0`** for deterministic output.

### Known limitations

| Limitation | Notes |
|---|---|
| In-memory sessions | History is lost on restart; capped at 5,000 sessions |
| Single FAISS instance | Instances do not share the index, so no horizontal scaling |
| Static knowledge base | New content needs a re-ingest |
| Word-based chunking | Answers spanning several chunks may be partly missed |
| No streaming | Replies arrive in one piece |

### Scaling path

- **Sessions** → Redis (a change confined to `session.py`).
- **Vector store** → Azure AI Search or Pinecone, shared across instances.
- **Hosting** → Azure Container Apps with autoscaling behind a load balancer.
- **Knowledge base** → scheduled re-scrape and re-embed with zero-downtime index swap.
- **Retrieval** → hybrid BM25 + FAISS and a cross-encoder re-ranker.
- **UX** → token streaming over Server-Sent Events.

## Author

**Padma Iyer** — [padmaiyer01@protonmail.com](mailto:padmaiyer01@protonmail.com)
