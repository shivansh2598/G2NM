# services/

Ingestion pipeline — converts markdown files into searchable vector embeddings.

## Structure

```
services/
├── ingest.py            # Entry point: python ingest.py --user {id}
├── chunker.py           # Splits markdown into overlapping chunks
└── embedder.py          # Embeds chunks via bge-small-en or text-embedding-3-small
```

## What ingest does

1. Reads all markdown from `users/{id}/knowledge/`
2. Chunks each document (respecting headings, paragraphs)
3. Tags chunks with structured metadata (skills, type, role)
4. Embeds chunks via the configured embedding model
5. Stores in ChromaDB collection `portfolio-{id}`

## Usage

```bash
# Local ChromaDB (embedded, SQLite-backed — default)
python services/ingest.py --user harshit

# Remote ChromaDB (Option C from deployment.md)
python services/ingest.py --user harshit --chroma-remote
```
