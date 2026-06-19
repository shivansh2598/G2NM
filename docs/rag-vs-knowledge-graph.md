# RAG vs Knowledge Graph — For a Portfolio Website

## Context

The portfolio knowledge base consists of ~10–20 markdown files:
- `about.md`, `resume.md`, `skills.md`
- Project writeups (`projects/*.md`)
- Work experience (`experience/*.md`)
- Blog posts (`blog/*.md`)

The question: **Does adding a knowledge graph on top of RAG provide meaningful improvement for this use case?**

## Plain RAG (Semantic Search)

### How it works

```
Documents → Chunk → Embed → Vector DB
Query → Embed → Similarity Search → Top-K Chunks → LLM Generation
```

### Strengths
- **Simple** — ~200–300 lines of pipeline code
- **Effective at small scale** — With 10–20 documents and `top_k=5`, nearly all relevant information is captured
- **Semantic understanding** — Embeddings capture meaning, not just keyword matching
- **Low maintenance** — Adding a new document = chunk it + embed it + insert it

### Weaknesses for this use case
- Can miss connections **spread across documents** (e.g., "Go" in resume, "Redis" in project writeup, "distributed" in skills — a query about "Go + databases" might miss the Redis link)
- Flat retrieval — no structured traversal of relationships

---

## RAG + Knowledge Graph (GraphRAG)

### How it works

```
Documents → Entity Extraction → Graph DB (Neo4j/NetworkX)
Query → Entity Recognition → Graph Traversal → Relevant Entities
       → Vector Search → Chunks → LLM Generation
```

Entities and relationships extracted from portfolio:
```
Shivi → WORKED_AT → Company A
Shivi → BUILT → Project B (Go, Redis, gRPC)
Company A → USED_TECH → Kubernetes
Project B → INVOLVES → distributed-systems
```

### Strengths
- **Multi-hop queries** — "What database experience does Shivi have with Go?" traverses person→project→technology→category
- **Explicit connections** — Relationships between entities are modeled, not inferred from embedding proximity
- **Better for complex knowledge bases** — 50+ documents with rich cross-references

### Weaknesses for this use case
- **Massive over-engineering for 10–20 documents**
- Requires entity extraction pipeline
- Requires graph database or in-memory graph (NetworkX)
- Graph + vector hybrid retrieval adds complexity
- Entity definitions must be manually maintained or extracted with another LLM call
- Deployment footprint grows significantly

---

## Decision: Structured Metadata (Middle Ground)

A third option bridges the gap — **metadata-filtered vector search**.

### How it works

During ingestion, each chunk is tagged with structured metadata:

```python
chunk = {
    "text": "Built a distributed cache in Go using gRPC and Redis...",
    "metadata": {
        "source": "projects/distributed-cache.md",
        "skills": ["Go", "gRPC", "Redis", "distributed-systems"],
        "type": "project",
        "role": "backend"
    }
}
```

At query time, retrieval combines semantic search with metadata filters:

```python
# Semantic search constrained to chunks tagged with "Go"
results = vector_db.search(
    query_embedding,
    top_k=5,
    filter={"skills": {"$in": ["Go"]}}
)
```

### Why this works for a portfolio site

| Concern | How metadata addresses it |
|---------|--------------------------|
| Cross-document connections | Tags explicitly link skills/projects/technologies |
| Missing Redis link | Tag query + semantic search captures it |
| Maintenance | Just update a few tags per markdown file |
| Complexity | Same vector DB, no graph DB needed |
| Scalability | If knowledge base grows, metadata structure supports migration to full GraphRAG |

### Trade-off

- Requires the author (you) to maintain ~5–10 tags per document
- Less powerful than a full graph for deeply nested queries
- But for this bounded scope, it's the pragmatic sweet spot

---

## Comparison Table

| Dimension | Plain RAG | RAG + Knowledge Graph | Metadata-Filtered RAG |
|-----------|-----------|----------------------|----------------------|
| Setup complexity | Low | High | Low-Medium |
| Cross-document connections | Weak | Strong | Good enough |
| Maintenance burden | Minimal | High (entity extraction) | Low (tags per doc) |
| Query latency | Low | Higher (graph + vector) | Same as plain RAG |
| Cost | Lowest | Higher (entity extraction LLM calls) | Same as plain RAG |
| Interview talking point | "Built RAG" | "Built GraphRAG" | "Evaluated GraphRAG, chose metadata-filtered RAG for scope" |
| Right for 10-20 docs? | ✅ | ❌ Overkill | ✅ Best fit |

---

## Conclusion

For a portfolio website with **10–20 documents**, **metadata-filtered RAG** is the right choice. It provides graph-like query capability through structured tags without the complexity and maintenance of a full knowledge graph.

The system architecture supports future migration to GraphRAG if the knowledge base scales to 50+ documents with rich cross-references — the metadata structure naturally extends to entity-relationship modeling when needed.

> "I evaluated GraphRAG vs. semantic RAG for this use case. Given the bounded scope, metadata-filtered vector search achieves the same retrieval quality with significantly lower complexity. The system is structured so that migrating to a hybrid graph+vector approach is straightforward if the knowledge base scales."
