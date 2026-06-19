# G2KM — Get to Know Me

A **data-driven portfolio terminal** — one codebase, any number of portfolio websites. Visitors interact with a terminal UI to explore your professional background, skills, projects, and personality. Backed by a RAG pipeline with DeepSeek.

## Quick start

```bash
git clone https://github.com/shivansh2598/G2KM.git
cd G2KM

# Copy the template and make it yours
cp -r users/_template users/your-name
# ... fill in users/your-name/knowledge/*.md with your content ...
# ... edit users/your-name/config.yaml ...

# Set your keys
cp .env.example .env
# ... edit .env with your actual keys ...

# Ingest your data
pip install -r requirements.txt
python services/ingest.py --user your-name

# Run locally
uvicorn app.main:app --reload

# Open http://localhost:3000
```

## Project structure

```
├── app/               # API layer + RAG pipeline
│   ├── api/           # Command handlers + ask endpoint
│   └── pipeline/      # Retrieval + generation logic
├── tui/               # Terminal UI frontend (xterm.js)
├── prompts/           # Versioned prompt templates (never hardcoded)
├── security/          # Input, content, and output guardrails
├── evaluation/        # Common engine tests (CI) + offline/online eval scripts
├── observability/     # Per-query tracing + cost tracking
├── services/          # Ingestion pipeline (chunk + embed + store)
├── users/             # Per-user data (gitignored, _template committed)
│   └── _template/     # Starter kit — copy this to create your portfolio
├── logs/              # Traces, feedback, guard rejection logs
├── docs/              # Architecture, security, deployment, and design docs
└── .env.example       # Environment variable reference
```

## Documentation

| Doc | Covers |
|-----|--------|
| [docs/architecture.md](docs/architecture.md) | System design, deployment model, component diagram |
| [docs/security.md](docs/security.md) | 6-layer guardrail model |
| [docs/rag-vs-knowledge-graph.md](docs/rag-vs-knowledge-graph.md) | Why metadata-filtered RAG over GraphRAG |
| [docs/knowledge-base.md](docs/knowledge-base.md) | The 15-document set and bucket structure |
| [docs/tui-commands.md](docs/tui-commands.md) | Command set, landing experience, feedback |
| [docs/deployment.md](docs/deployment.md) | Docker options, cloud hosting, step-by-step deploy |
| [docs/observability.md](docs/observability.md) | Tracing, cost tracking, privacy, dashboards |
