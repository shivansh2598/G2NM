# Security & Guardrails

## Threat Model

The portfolio website exposes an LLM-backed endpoint to the public internet. Without guardrails, anyone can use it as a free AI proxy for purposes unrelated to the portfolio — draining tokens and cost.

**Goal:** Ensure the system ONLY answers questions about the portfolio owner's professional background and rejects all other queries at minimal cost.

## Security Approach

We had a discussion around implementing a multi-layer security model to protect the system from off-topic queries and ensure safe usage. The exact combination of layers will be determined to provide the best balance of security and cost-efficiency.

---

## Defense Layers Under Consideration

The guiding principle: **reject off-topic queries at the cheapest layer possible** to avoid unnecessary LLM calls.

---

### Layer 1: Input Heuristics (Cost: $0)

Quick, pre-embedding checks that catch obvious misuse before any model call.

- Reject queries exceeding a reasonable character limit (e.g., 300 chars) — no one needs that length to ask about a person
- Detect code blocks (```) — strong signal of misuse
- Pattern-match common prompt injection phrases (`ignore`, `forget`, `pretend`, `act as`, `system:`)

**Catches:** Prompt injection attempts, code requests, clearly off-topic long prompts.

---

### Layer 2: Similarity Score Threshold (Cost: ~1 embedding call)

After embedding the user query, check if it's semantically similar to any portfolio content before proceeding to retrieval and generation.

- Run similarity search with `top_k=1`
- If the best match score falls below a tuned threshold, return the refusal response immediately
- No DeepSeek call occurs for queries semantically distant from portfolio content

**Threshold tuning is critical:** too strict → legitimate questions fail; too loose → off-topic queries leak through.

**Catches:** Any query semantically distant from portfolio content (homework, general knowledge, coding problems, etc.).

---

### Layer 3: Rate Limiting (Cost: $0)

Caps financial exposure from volume abuse regardless of query content.

| Limit | Value | Scope |
|-------|-------|-------|
| Requests per minute | 10 | Per IP |
| Concurrent requests | 3 | Per session |
| Daily requests | ~200 | Global (adjust based on traffic) |

Can be implemented at the application level (`slowapi` for Python/FastAPI, `express-rate-limit` for Node.js) or at the infrastructure level (Cloudflare, reverse proxy).

**Catches:** Token draining attacks, bots, excessive usage from any single source.

---

### Layer 4: System Prompt Hardening (Cost: $0, part of generation)

The system prompt strictly constrains the model's behavior. This is a last line of defense, not the first — prompt injection can bypass system prompts.

The prompt will:
- Define the assistant's scope: only the portfolio owner's professional background
- Provide a strict refusal message for out-of-scope queries
- Explicitly list disallowed topics (coding problems, general knowledge, homework, etc.)
- Instruct the model to never acknowledge or comply with prompt injection

---

### Layer 5: Topic Classifier (Cost: 1 small/cheap LLM call)

A separate, lightweight classification step that runs before the main generation, or in parallel.

- Small prompt: "Is this query about {name}'s professional background? YES/NO"
- Can use a tiny local model (quantized Llama via Ollama) for zero per-query cost, or DeepSeek for minimal cost
- If classified OFF_TOPIC → return refusal, no main generation

**Catches:** Semantic violations that pass the similarity threshold but aren't actually about the portfolio owner.

---

### Layer 6: Output Validation — LLM Judge (Cost: 1 LLM call)

After generation, verify the response stayed on-topic. This uses LLM-as-judge: one model evaluates another model's output.

- Prompt: "Does this response discuss {name}'s professional background? YES/NO"
- If NO → replace the response with the refusal message
- Catches cases where the model ignored the system prompt despite all input guards

This is the same pattern used in production by RAGAS, Guardrails AI, and content safety filters at major AI companies.

---

## Layered Flow

```
User Query
    │
    ▼
[Layer 1: Input Heuristics] ── FAIL ──► OFF_TOPIC ($0)
    │ PASS
    ▼
[Layer 2: Similarity Threshold] ── FAIL ──► OFF_TOPIC (~1 embedding)
    │ PASS
    ▼
[Layer 3: Rate Limiting] ── FAIL ──► 429 ($0)
    │ PASS
    ▼
[Layer 4: System Prompt + Generation] ──► DeepSeek Response
    │
    ▼
[Optional Layer 5: Topic Classifier]
    │
    ▼
[Optional Layer 6: Output Judge] ── FAIL ──► Replace with refusal
    │ PASS
    ▼
Stream response to TUI
```

---

## Decision Status

The exact combination of layers is still being finalized. The current thinking:

- **Layers 1–4** provide strong coverage at minimal cost and will likely form the foundation
- **Layers 5–6** add stronger assurance but increase per-query cost with additional LLM calls
- The final decision will depend on monitoring real-world traffic patterns and identifying which types of misuse slip past the initial layers

The architecture will be designed to support all six layers, with the ability to enable/disable layers 5–6 based on observed needs.
