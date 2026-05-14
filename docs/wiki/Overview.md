# RAG System with Observability — Overview

A production-ready Retrieval Augmented Generation (RAG) system that lets users ask natural-language questions over a document knowledge base and receive cited, grounded answers. The project ships with full observability, an automated evaluation suite, and a React-based chat UI.

---

## Key Features

| Area | What you get |
|------|-------------|
| **Document Ingestion** | PDF and text file processing with token-aware chunking |
| **Vector Search** | Dense retrieval powered by Qdrant |
| **LLM Generation** | OpenAI API with automatic local-model fallback (Ollama) |
| **Observability** | OpenTelemetry tracing, Prometheus metrics, Grafana dashboards |
| **Evaluation** | Automated precision@k, hallucination detection, BLEU/ROUGE |
| **Caching** | Multi-layer Redis caching (embeddings, retrieval, LLM responses) |
| **Resilience** | Circuit breaker pattern with exponential backoff |
| **UI** | React + Tailwind chat interface with source citations and feedback |
| **Privacy** | PII redaction, API-key auth, rate limiting |

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend | Python, FastAPI |
| Frontend | React, Tailwind CSS |
| Vector DB | Qdrant (self-hosted via Docker) |
| LLM | OpenAI `gpt-3.5-turbo` (primary), Ollama (fallback) |
| Embeddings | OpenAI `text-embedding-3-small`, sentence-transformers (fallback) |
| Caching | Redis 7 |
| Orchestration | LangChain |
| Observability | OpenTelemetry, Prometheus, Grafana |
| Containerisation | Docker Compose |

---

## Quick Start

### Prerequisites

- Python 3.9+
- Docker Desktop
- An OpenAI API key

### 1. Clone & install

```bash
git clone <repo-url>
cd rag_observability
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 2. Configure environment

```bash
cp env.template .env
# Open .env and set your OPENAI_API_KEY
```

### 3. Start infrastructure services

```bash
docker compose up -d
```

This brings up four services:

| Service | Port | Purpose |
|---------|------|---------|
| **Qdrant** | 6333 | Vector database for document embeddings |
| **Redis** | 6379 | Caching layer (embeddings, retrieval, LLM responses) |
| **Prometheus** | 9090 | Metrics collection |
| **Grafana** | 3000 | Dashboards and visualization |

### 4. Start the application

```bash
# Backend API
PYTHONPATH=. uvicorn api.main:app --host 0.0.0.0 --port 8080

# Frontend (separate terminal)
cd ui && npx vite --host 0.0.0.0 --port 5173
```

### 5. Install the frontend (first time only)

```bash
cd ui && npm install
```

---

## Project Structure

```
rag_observability/
├── api/                  # FastAPI REST endpoints
├── src/
│   ├── config/           # Centralized settings (env-driven)
│   ├── ingest/           # Document loading & chunking
│   ├── retrieval/        # Vector search & context compression
│   ├── generator/        # LLM calls & prompt versioning
│   ├── eval/             # Evaluation framework
│   ├── observability/    # Tracing & metrics
│   ├── cache/            # Redis caching layer
│   └── utils/            # PII redaction, circuit breaker, helpers
├── ui/                   # React frontend
├── tests/                # Pytest test suite
├── data/
│   ├── raw/              # Source documents (PDF, text)
│   └── processed/        # Chunked JSONL output
├── prompts/              # Versioned prompt templates (v1, v2, …)
├── dashboards/           # Grafana dashboard JSON
├── docker/               # Docker helper configs
├── docs/                 # Documentation & wiki
└── scripts/              # CLI utilities
```

---

## Main Components

### Ingestion Pipeline (`src/ingest/`)
Loads PDFs and text files, splits them into token-aware chunks (default 500 tokens, 50-token overlap), and writes JSONL output. Each chunk carries metadata such as `doc_id`, `chunk_id`, and `token_count`.

### Retrieval (`src/retrieval/`)
Generates query embeddings, performs dense vector search against Qdrant (top-k, default 10), and compresses results using score filtering (cosine similarity ≥ 0.7). Optional cross-encoder reranking and hybrid BM25 search are available.

### Generator (`src/generator/`)
Formats retrieved context into versioned prompt templates, calls the LLM, and returns a structured response with the answer, source citations, token usage, and cost estimate. Falls back to a local Ollama model when the OpenAI circuit breaker trips.

### Caching (`src/cache/`)
Three-tier Redis cache: embeddings (24 h TTL), retrieval results (1 h), and LLM responses (6 h). Targets a ≥ 70 % cache-hit rate for repeated queries.

### Observability (`src/observability/`)
OpenTelemetry spans cover every pipeline stage. Prometheus exports latency histograms, token counters, cost gauges, and cache-hit rates. A pre-built Grafana dashboard visualises all key metrics.

### Evaluation (`src/eval/`)
Computes precision@k (k = 1, 3, 5), retrieval recall, BLEU/ROUGE, and LLM-judged faithfulness. Designed to run in CI with configurable thresholds (e.g., hallucination rate < 5 %).

### Configuration (`src/config/settings.py`)
All tuneable parameters (model names, cache TTLs, retrieval thresholds, rate limits, etc.) are managed through environment variables loaded via Pydantic `BaseSettings`. See `env.template` for the full list.

### API & UI (`api/` and `ui/`)
FastAPI serves the REST endpoints (`/api/query`, `/api/feedback`, `/api/metrics`, `/api/trace/{trace_id}`). The React frontend provides a chat interface with inline source citations and a thumbs-up/down feedback loop.

---

> For the full deep-dive wiki, see <https://deepwiki.com/Gourab-18/rag_observability>.
> For the detailed implementation plan, see [`plan.md`](../../plan.md).
> For setup troubleshooting, see [`docs/SETUP.md`](../SETUP.md) (if available) or [`docs/spec.md`](../spec.md).
