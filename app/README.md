# app/

API layer and RAG pipeline. Single-user per deployment — no multi-tenant routing.

## Structure

```
app/
├── main.py              # FastAPI app entry point
├── api/
│   ├── commands.py      # Command handlers (resume, projects, skills, etc.)
│   └── ask.py           # RAG query endpoint (the /ask route)
└── pipeline/
    ├── retrieval.py     # Embed query → similarity search → top-k chunks
    └── generation.py    # Assemble prompt → DeepSeek → stream response
```

## Notes

- Commands fetch content directly from markdown files — no LLM cost.
- `ask` is the only endpoint that hits the full RAG pipeline.
- USER_ID is read from env at startup, not per-request. One deployment = one user.
