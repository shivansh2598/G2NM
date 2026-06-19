# Observability

Every `ask` query is traced end-to-end. This serves three purposes: debugging, cost management, and understanding what visitors actually want to know.

## What's traced per query

```json
{
  "timestamp": "2026-06-19T14:30:00Z",
  "query_id": "q_abc123",
  "user_query": "what projects has Shivi built with Go?",
  "ip_hash": "a1b2c3...",
  "stages": {
    "guard_heuristics": { "passed": true, "latency_ms": 0.1 },
    "guard_similarity": { "passed": true, "score": 0.87, "threshold": 0.5, "latency_ms": 45 },
    "rate_limiter": { "allowed": true, "remaining_tokens": 8 },
    "embedding": { "model": "bge-small-en", "latency_ms": 12, "input_tokens": 8 },
    "retrieval": { "top_k": 5, "chunks_found": 5, "scores": [0.87, 0.82, 0.78, 0.71, 0.65], "latency_ms": 3 },
    "generation": {
      "model": "deepseek-chat",
      "prompt_tokens": 450,
      "completion_tokens": 120,
      "latency_ms": 1200
    },
    "output_guard": { "passed": true, "latency_ms": 0 }
  },
  "cost": {
    "embedding": 0.0001,
    "generation": 0.0008,
    "total": 0.0009
  },
  "total_latency_ms": 1260
}
```

## Cost per query breakdown

| Component | Pricing basis | Estimated cost per query |
|-----------|--------------|--------------------------|
| Embedding (bge-small-en) | Local — $0 | $0.0000 |
| Embedding (text-embedding-3-small) | $0.02 / 1M tokens | ~$0.0001 |
| DeepSeek generation | ~$0.14 / 1M input tokens, ~$0.28 / 1M output tokens | ~$0.0005–$0.002 |
| **Total (local embedding)** | | **~$0.0005–$0.002** |
| **Total (OpenAI embedding)** | | **~$0.0006–$0.002** |

At $0.002/query, 200 queries/day = $0.40/day = **~$12/month**. DeepSeek makes this viable for a public portfolio.

## What you learn from the traces

| Insight | From |
|---------|------|
| **What are visitors asking?** | `user_query` field — can see if recruiters ask about specific skills, projects, or culture |
| **What gets rejected?** | Guard failures — see patterns in off-topic queries, tune similarity threshold |
| **Where's the latency?** | Per-stage `latency_ms` — is DeepSeek slow, or is embedding the bottleneck? |
| **How much is this costing?** | Per-query cost + daily/weekly aggregates |
| **Are retrievals any good?** | Similarity scores — if top chunks score 0.5, something's wrong with the knowledge base |
| **Are prompts too fat?** | `prompt_tokens` — if you're stuffing too many chunks, trim the prompt template |

## Storage (v1)

Traces are written as structured JSON lines to log files. For a portfolio site (~200 queries/day), this is ~5MB/month.

```
logs/
├── traces-2026-06-19.jsonl    # Per-query traces
├── daily-costs.csv             # Rolling cost summary
└── guard-rejections.jsonl      # Queries blocked by each guard layer
```

No external observability service required — plain files work at this scale.

## Storage (v2 — if traffic grows)

Graduate to a lightweight SQLite database or Prometheus + Grafana for dashboards.

## Privacy

Queries are logged without identifying information — IP addresses are hashed, no cookies or session IDs are stored. This is a public portfolio terminal; visitors should know their queries may be logged for quality improvement, similar to how a website logs page views. Add a brief note in the TUI welcome message or provide a `privacy` command.

## Dashboard (optional v2)

A lightweight admin view at `/admin` (password-protected) showing:
- Queries over time (sparkline)
- Top 10 most-asked questions
- Daily cost
- Guard rejection rate
- Average latency

Not required for v1 — the JSONL logs are grep-able and sufficient for early traffic analysis.

## Integration with evaluations

Evaluation results (see [architecture.md](architecture.md)) are logged alongside query traces for correlation — if a query scored poorly on faithfulness, the trace shows exactly what chunks were retrieved and what the model generated.
