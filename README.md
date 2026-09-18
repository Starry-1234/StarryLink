# StarryLink

> An AI-powered e-commerce customer service agent built on LangGraph + FastAPI + Milvus + MySQL. Supports order / logistics queries, knowledge-base Q&A, intent dispatch, refund workflows, knowledge harvesting from conversations, and topic classification.

StarryLink is Starry's personal project.

## Features

- **Multi-turn chat** with streaming, structured extraction, and coreference resolution
- **5 built-in tools**: order, logistics, FAQ, product, refund ticket creation
- **Hybrid retrieval** (BM25 + vector + RRF + reranker) over a self-managed knowledge base
- **LangGraph stateful agent** with intent dispatch and refund interrupt/resume
- **Self-hosted Langfuse** for LLM trace + cost accounting (data stays on your machine)
- **Flywheel**: mine Q&A pairs from conversations to grow the knowledge base
- **Topic classifier** with ONNX inference

## Tech Stack

| Layer | Tech |
|---|---|
| Web | FastAPI + Uvicorn |
| Agent | LangGraph + LangChain |
| LLM clients | OpenAI-compatible (chat / embed / rerank, no gateway) |
| Storage | MySQL 8 + Milvus Standalone + MinIO + etcd |
| Observability | Langfuse v3 (self-hosted, OTLP) |
| Package manager | uv |
| Container | Docker Compose v2 |

## Quick Start

```bash
# 1. Configure (fill CHAT_* / EMBED_* / RERANK_* in .env)
cp .env.example .env
$EDITOR .env

# 2. Data services
docker compose up -d

# 3. Seed business data
make seed

# 4. Build knowledge base
make kb-build
make kb-vectorize

# 5. App + MCP servers
make dev
```

Open <http://localhost:8000> for the chat UI.

To start Langfuse (for observability):

```bash
make langfuse-up
# Open http://localhost:3000
```

## Project Layout

| Path | What lives here |
|---|---|
| `app/api/` | HTTP endpoints (chat, agent, KB, review, eval, cost) |
| `app/graph/` | LangGraph state machine (nodes, routing, build) |
| `app/core/` | Single-purpose modules (LLM client, retrieval, intent, coref, summary, confidence, observability) |
| `app/kb/` | Knowledge base: chunking, embedding, MySQL+Milvus dual-write, dedup, mining |
| `app/tools/` | Tool system: built-in `@tool`, MCP client, registry, executor |
| `app/db/` | SQLAlchemy models and repositories |
| `app/static/` | Frontend HTML pages (SSR, no build step) |
| `mcp_servers/` | Two business MCP servers (logistics, aftersales) |
| `sql/` | Per-chapter DDL / seed SQL, applied at first boot |
| `scripts/` | Build / eval / tune / observe offline jobs |
| `docs/superpowers/` | Specs and plans |
| `primer/` | Primer examples (LLM, agent basics) — independent, share `.env` |

## Pages

| URL | Page |
|---|---|
| `/` | Chat |
| `/kb` | Knowledge base ingest |
| `/review` | Flywheel review queue |
| `/observability` | Cost & observability dashboard |
| `/topics` | Topic distribution |
| `/acceptance` | Classifier acceptance |

## Ports

| Port | Service |
|---|---|
| 8000 | App |
| 8101 / 8102 | MCP servers (logistics / aftersales) |
| 8110 | Topic classifier (after `make classifier-up`) |
| 3000 | Langfuse web (after `make langfuse-up`) |
| 19530 | Milvus |

## Development

```bash
make test         # unit tests
make eval         # chat eval
make eval-agent   # tool calling
make kb-build     # KB build
make kb-vectorize # vectorize KB
make eval-rag     # retrieval eval
make eval-ch05    # LangGraph
make eval-ch06    # interrupt
make eval-ch07    # context
make eval-ch08    # tools
make cost-report  # cost
make ch10-corpus  # ch10 corpus
make ch10-train   # ch10 train
make ch10-eval    # ch10 eval
```

## License

MIT — see [LICENSE](LICENSE).

## Contact

GitHub: [@Starry-1234](https://github.com/Starry-1234)
