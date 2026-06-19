# prompts/

Prompt templates live here — never hardcoded inline. In the spirit of the LinkedIn post: versioned, typed, registered.

## Files

```
prompts/
├── system.yaml          # System prompt — defines assistant scope, refusal message
├── judge.yaml           # LLM judge prompt — for output validation (Layer 6)
└── classifier.yaml      # Topic classifier prompt — for Layer 5 (optional)
```

## Template variables

All prompts use `{name}` and `{pronouns}` from `users/{USER_ID}/config.yaml`.

```yaml
# Example: system.yaml
system: |
  You are a RAG assistant representing {name}'s professional portfolio.
  You ONLY answer questions about {name}'s experience, skills, projects,
  and professional background using the provided context.
  If a question is not about {name}, respond with:
  "I can only answer questions about {name}'s professional background..."
```
