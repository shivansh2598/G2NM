# observability/

Per-stage tracing, cost tracking, guard analytics. Know what's happening before something breaks.

## Structure

```
observability/
├── tracer.py            # Per-query JSON trace (all stages, latencies, scores)
└── cost_tracker.py      # Token counting + cost estimation per query
```

## Logs (written to logs/)

```
logs/
├── traces-YYYY-MM-DD.jsonl     # Per-query traces
├── daily-costs.csv              # Rolling cost summary
├── guard-rejections.jsonl       # Queries blocked by each guard
└── feedback-YYYY-MM-DD.jsonl    # User feedback from `feedback` command
```

## Trace format

See [docs/observability.md](../docs/observability.md) for the full trace schema, cost breakdown, and v2 plans.
