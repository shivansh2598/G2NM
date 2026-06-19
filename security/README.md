# security/

Input, content, and output guardrails. Three guards, not one.

## Structure

```
security/
├── heuristics.py        # Layer 1: Input heuristics (char limit, code detection, injection patterns)
├── similarity.py        # Layer 2: Similarity score threshold (reject below threshold)
├── rate_limiter.py      # Layer 3: Rate limiting (per-IP, per-session, global)
├── classifier.py        # Layer 5 (opt): Topic classifier — ON_TOPIC / OFF_TOPIC
└── output_judge.py      # Layer 6 (opt): LLM judge for output validation
```

## Architecture

See [docs/security.md](../docs/security.md) for the full 6-layer model and decision status.
