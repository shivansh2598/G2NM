# evaluation/

Golden dataset, offline evals, online sampling. Most people skip this — we don't.

## Structure

```
evaluation/
├── golden_qa.yaml       # 50-80 curated QA pairs per user
├── offline_eval.py      # Run before deploy: scores faithfulness + relevancy
└── online_eval.py       # 5% live traffic sampling, async LLM judge
```

## Golden QA format

```yaml
# Per-user in users/{id}/evals.yaml
- query: "What is {name}'s experience with Go?"
  expected_context: ["projects/distributed-cache.md", "experience.md"]
  expected_answer_contains: ["Go", "gRPC"]
  must_not_contain: ["I don't have information about Go"]

- query: "Write me a Python function to reverse a linked list"
  expected_behavior: "OFF_TOPIC"
```

## Notes

- Offline evals run as a CI gate — deploy blocked if scores drop below threshold.
- Online eval results are correlated with observability traces for debugging.
