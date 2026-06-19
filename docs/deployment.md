# Deployment

How a collaborator (e.g., Harshit, Shashwat) goes from cloning the repo to having their own live portfolio site — without writing any code.

## Overview

```
GitHub Repo (shared engine)
    → Clone
    → Add personal markdown files (users/{name}/knowledge/*.md)
    → Configure (config.yaml + .env)
    → Ingest (chunk + embed → vector DB)
    → Deploy (container → cloud)
    → Live site at {name}.portfolio.app
```

Each person deploys their own **independent instance** with their own database, API keys, and hosting. The only shared artifact is the engine code in the GitHub repository.

---

## Step-by-step flow

### Step 1: Clone the repo

```bash
git clone https://github.com/shivi/portfolio-orchestrator.git
cd portfolio-orchestrator
```

### Step 2: Create personal data directory

```bash
cp -r users/_template users/{your-name}
```

The `_template/` directory contains stub markdown files and an example config. Personal data in `users/` is gitignored — it will never be pushed to GitHub.

```
users/{your-name}/
├── config.yaml          # Edit this
├── evals.yaml           # Optional: populate with your QA pairs
└── knowledge/           # Fill all 15 .md files with your content
    ├── experience.md
    ├── skills.md
    ├── education.md
    ├── projects.md
    ├── how-i-work.md
    ├── tech-hot-takes.md
    ├── day-in-the-life.md
    ├── talks-writing.md
    ├── open-source.md
    ├── opportunities.md
    ├── travel.md
    ├── hobbies.md
    ├── media.md
    ├── values.md
    └── fun-facts.md
```

### Step 3: Configure

Edit `users/{your-name}/config.yaml`:

```yaml
id: harshit
name: Harshit
pronouns: he/him
domain: harshit.portfolio.app
```

Create `.env` with your credentials:

```bash
DEEPSEEK_API_KEY=sk-your-deepseek-key
EMBEDDING_MODEL=bge-small-en        # or text-embedding-3-small
OPENAI_API_KEY=sk-your-openai-key   # only if using OpenAI embeddings
USER_ID=harshit
```

### Step 4: Ingest (build embeddings)

```bash
pip install -r requirements.txt
python ingest.py --user harshit
```

What this does:
- Reads all markdown files from `users/harshit/knowledge/`
- Chunks each document
- Generates embeddings via the configured embedding model
- Stores vectors in a local ChromaDB (`chroma_data/` directory)
- Tags each chunk with structured metadata (skills, type, role)

---

## Deployment Options

Three approaches, from simplest to most decoupled.

### Option A: Docker with embedded Chroma (recommended for v1)

Everything runs in one container. ChromaDB is embedded (SQLite-backed). No external database needed.

**Architecture:**

```
┌──────────────────────────────┐
│  Docker Container             │
│  ┌──────────┐  ┌───────────┐ │
│  │  FastAPI  │  │ ChromaDB  │ │
│  │  App      │──│ (embedded)│ │
│  └──────────┘  └───────────┘ │
│       chroma_data/ on disk   │
└──────────────────────────────┘
         │
    DeepSeek API (remote)
```

**Dockerfile:**

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app/ ./app/
COPY ingest.py .
COPY chroma_data/ ./chroma_data/

EXPOSE 3000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "3000"]
```

**Deploy:**

```bash
# Build image (embeddings already ingested locally)
docker build -t harshit-portfolio .
docker tag harshit-portfolio harshit/harshit-portfolio:latest
docker push harshit/harshit-portfolio:latest

# On EC2 (or any cloud VM)
ssh ec2-user@your-server
docker pull harshit/harshit-portfolio:latest
docker run -d -p 80:3000 \
  -e DEEPSEEK_API_KEY=sk-xxx \
  -e USER_ID=harshit \
  harshit-portfolio:latest
```

**Hosting options:**
- AWS EC2 micro (~$5/month)
- Fly.io (free tier available)
- Railway ($5/month)
- DigitalOcean droplet ($6/month)
- Render (free tier with cold starts)

**Pros:**
- Zero external services beyond the VM
- One command to deploy
- No separate database to manage

**Cons:**
- Must rebuild and redeploy container to update portfolio data

---

### Option B: Build embeddings during Docker build

Ingestion happens inside the Dockerfile — no local ingestion step needed. Markdown files must be present at build time.

**Dockerfile:**

```dockerfile
# Stage 1: Build embeddings
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY users/harshit/ ./users/harshit/
COPY ingest.py .
RUN python ingest.py --user harshit

# Stage 2: Runtime (slim image, no build tools)
FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /app/chroma_data/ ./chroma_data/
COPY --from=builder /usr/local/lib/python3.12/site-packages/ /usr/local/lib/python3.12/site-packages/
COPY app/ ./app/
COPY requirements.txt .
RUN pip install -r requirements.txt
EXPOSE 3000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "3000"]
```

**Deploy:**

```bash
docker build -t harshit-portfolio .   # Ingestion + packaging in one step
docker push harshit/harshit-portfolio:latest
# ... deploy same as Option A
```

**Pros:**
- Fully reproducible — `docker build` is the only command
- No manual ingestion step

**Cons:**
- Embedding model must run in Docker build (needs network for API-based, or CPU/RAM for local model)
- Markdown files must not be gitignored for this to work (local clone only)

---

### Option C: Hosted ChromaDB + decoupled ingestion (production-grade)

App and vector database are separate services. Ingestion populates hosted ChromaDB once; the app connects remotely.

**Architecture:**

```
┌──────────────────────┐
│  Hosted ChromaDB      │
│  Collection:          │
│  portfolio-harshit    │
└──────────┬───────────┘
           │ connection string
┌──────────▼───────────┐
│  App Container        │
│  (FastAPI only)       │
└──────────┬───────────┘
           │
      DeepSeek API
```

**.env:**

```bash
DEEPSEEK_API_KEY=sk-xxx
CHROMA_HOST=https://chroma.harshit.com
CHROMA_API_KEY=chroma-api-key
USER_ID=harshit
```

**Ingestion (run once):**

```bash
python ingest.py --user harshit --chroma-remote
```

**Deploy app (no DB inside):**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app/ ./app/
COPY requirements.txt .
RUN pip install -r requirements.txt
EXPOSE 3000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "3000"]
```

**Pros:**
- Update embeddings without redeploying the app
- App and data decoupled
- Scales independently

**Cons:**
- Two services to manage
- Hosted ChromaDB costs extra (or self-hosted on a second VM)

---

## Recommendation

| Scenario | Use |
|----------|-----|
| **v1 — get live fast** | Option A (embedded Chroma, one container) |
| **Stable, want cleaner CI** | Option B (ingest during Docker build) |
| **Multiple collaborators, frequent updates** | Option C (hosted ChromaDB, decoupled) |

For the initial launch, **Option A** is the pragmatic choice. Graduate to Option C if data updates become frequent and redeploying containers becomes painful.

---

## What collaborators never have to do

- ❌ Write any code (unless contributing to the engine)
- ❌ Build a TUI
- ❌ Configure DeepSeek prompts
- ❌ Build a RAG pipeline
- ❌ Implement security guardrails
- ❌ Fork the repo and maintain a separate codebase
- ❌ Push personal data to GitHub

## What collaborators bring themselves

| Resource | Provided by |
|----------|-------------|
| Markdown content (15 docs) | Collaborator |
| config.yaml | Collaborator |
| DeepSeek API key | Collaborator |
| Embedding model (local or API key) | Collaborator |
| Cloud hosting | Collaborator |
| Domain | Collaborator |
| Vector database | Option A: embedded (free) / Option C: collaborator's own |
| TUI, RAG pipeline, guardrails, prompts, eval framework | The repo |
