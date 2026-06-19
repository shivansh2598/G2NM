# Architecture

## Overview

A **data-driven portfolio terminal** — one codebase, any number of portfolio websites. Each person brings their own markdown files and config; the engine stays the same.

### How it works

Each person clones the repo, adds their data, and deploys their own site.

```
┌──────────────────────────────┐
│     GitHub Repository        │
│   (shared engine code only)  │
└────────────┬─────────────────┘
             │ clone + add personal data
   ┌─────────┼─────────┐
   ▼         ▼         ▼
┌──────┐ ┌──────┐ ┌──────┐
│Shivi │ │Harsh │ │Shash │   ← Each person's clone
│ data │ │ data │ │ data │
│ creds│ │ creds│ │ creds│
└──┬───┘ └──┬───┘ └──┬───┘
   ▼        ▼        ▼
┌──────┐ ┌──────┐ ┌──────┐
│  DB  │ │  DB  │ │  DB  │   ← Independent databases
└──────┘ └──────┘ └──────┘
   ▼        ▼        ▼
shivi.   harshit.  shashwat.
portfolio portfolio portfolio
.app      .app      .app           ← Independent deployments
```

**Key implications:**
- No auth required — each person deploys their own instance
- No PR review for personal data — markdown files live in each person's clone, not the shared repo
- PR review only for engine code changes that all collaborators benefit from
- `users/` is in `.gitignore`; the repo ships `users/_template/` as a starting point
- Each person provides their own DB credentials and API keys via `.env`

**Key principle:** The code doesn't know about Shivi or Harshit or Shashwat. It knows about a generic "user" who has markdown files, a config file, and a vector collection. The same codebase produces portfolio sites for anyone who clones it.

## Motivation

- **Code reuse** — One repository, any number of portfolio sites. No forking, no duplicating logic.
- **Collaboration** — Multiple people can work on the engine while each powers their own site.
- **Demonstration** — Proves data-driven architecture and RAG engineering.
- **Memorable** — Terminal-style UI stands out; RAG backend proves deep AI engineering skill.

## System Components

```
┌───────────────────────────────────────────────────────┐
│                 Browser (TUI)                         │
│            xterm.js / custom React shell              │
└──────────────────────┬────────────────────────────────┘
                       │ HTTP / WebSocket
                       ▼
┌───────────────────────────────────────────────────────┐
│                  API Layer                             │
│              FastAPI / Hono / Next.js                  │
│                                                       │
│  ┌─────────┐  ┌───────────┐  ┌──────────────────┐   │
│  │  Input  │  │ Retrieval │  │   Generation     │   │
│  │ Guards  │──│ Pipeline  │──│   (DeepSeek)     │   │
│  └─────────┘  └───────────┘  └──────────────────┘   │
└──────────────────────┬────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────┐
│  Embedded ChromaDB (one collection per deployment)    │
│  Collection: portfolio-{user_id}                      │
│  ┌──────────────┐                                    │
│  │ chroma_data/ │  ← SQLite-backed, lives in container│
│  └──────────────┘                                    │
└───────────────────────────────────────────────────────┘
                       │
                       ▼
┌───────────────────────────────────────────────────────┐
│  users/{id}/                                          │
│  ├── config.yaml                                      │
│  ├── knowledge/*.md                                   │
│  └── evals.yaml                                       │
└───────────────────────────────────────────────────────┘
```

## What's in the deployment

| Layer | Notes |
|-------|-------|
| TUI frontend | xterm.js / custom React shell |
| API layer | FastAPI / Hono / Next.js — serves both commands and `ask` |
| Input guards | Heuristics, rate limiting, similarity threshold (see [security.md](security.md)) |
| RAG pipeline | Chunk → embed → search → generate — identical logic for every deployment |
| LLM | DeepSeek — token-efficient generation |
| Embedding model | `bge-small-en` (local) or `text-embedding-3-small` (OpenAI) — per-deployment choice |
| ChromaDB | Embedded, SQLite-backed — one collection per deployment, lives inside the container |
| Observability | Per-query JSONL traces — latency, token count, cost, guard decisions (see [observability.md](observability.md)) |
| User data | `users/{id}/config.yaml`, `users/{id}/knowledge/*.md`, `users/{id}/evals.yaml` |

Since each deployment is independent, there is no "shared vs. per-user" distinction at runtime. The codebase is shared via GitHub; at deploy time, each container is self-contained with one user's data.

## User Configuration

Each deployment has one user, identified by `USER_ID` in `.env`:

```bash
# .env
USER_ID=shivi
DEEPSEEK_API_KEY=sk-xxx
```

The app loads `users/{USER_ID}/config.yaml` at startup:

```yaml
# users/shivi/config.yaml
id: shivi
name: Shivi
pronouns: they/them
domain: shivi.portfolio.app
system_prompt_overrides: {}  # optional
```

## Setup (for each person)

See [deployment.md](deployment.md) for the full step-by-step flow. The short version: clone the repo, copy `users/_template`, fill in 15 markdown files, configure `.env`, run `python ingest.py`, deploy. No code changes, no forking.

## Knowledge Base

Each deployment has one user's 15 markdown files, chunked and stored with structured metadata. Full document list and bucket structure at [knowledge-base.md](knowledge-base.md).

## Query Flow

Two paths: **commands** (`resume`, `projects`, etc.) fetch content directly — no LLM cost. **`ask <question>`** runs the full RAG pipeline: input guards → embed → similarity search → top-k chunks → DeepSeek generates → stream response. The `USER_ID` env var determines whose data is queried.

See [security.md](security.md) for the guard layers, [rag-vs-knowledge-graph.md](rag-vs-knowledge-graph.md) for the metadata-filtered retrieval approach.

## Evaluation Strategy

Two layers: **common** (committed, tests the engine in CI) and **per-user** (gitignored, tests content accuracy before deploy). Offline evals run both via LLM-as-judge scoring. Online evals sample 5% of live traffic. Results correlated with query traces (see [observability.md](observability.md)).

## AI Model Choices

| Stage | Component | Model | Rationale |
|-------|-----------|-------|-----------|
| **Ingestion** | Embedding | `bge-small-en` (local, free) or `text-embedding-3-small` (OpenAI, cheap) | Embed documents into vectors for similarity search |
| **Query embedding** | Embedding | Same as ingestion | Convert user query to vector |
| **Generation** | LLM | **DeepSeek** | Token-efficient, low cost, strong instruction following |

**Why DeepSeek:** DeepSeek uses fewer tokens than GPT or Claude for equivalent outputs, keeping per-query costs minimal for publicly exposed sites.

## Related Docs

| Doc | Covers |
|-----|--------|
| [security.md](security.md) | Input/output guardrails, rate limiting, LLM judge |
| [rag-vs-knowledge-graph.md](rag-vs-knowledge-graph.md) | Metadata-filtered RAG vs. GraphRAG decision |
| [knowledge-base.md](knowledge-base.md) | The 15-document set and bucket structure |
| [tui-commands.md](tui-commands.md) | TUI command set, landing experience, command vs. RAG balance |
| [deployment.md](deployment.md) | Step-by-step collaborator deployment flow, Docker options, hosting |
| [observability.md](observability.md) | Per-query tracing, cost tracking, guard analytics, v1/v2 storage |
