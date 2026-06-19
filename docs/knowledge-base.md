# Knowledge Base — Document Set

15 markdown documents powering the RAG-based portfolio. Organized into four buckets.

## Resume Essentials (3 docs)

| # | Doc | What it covers |
|---|-----|----------------|
| 1 | `experience.md` | Roles, companies, dates, key achievements per role |
| 2 | `skills.md` | Languages, frameworks, tools — with proficiency levels |
| 3 | `education.md` | Degrees, certifications, relevant coursework |

## Deep Professional (6 docs)

| # | Doc | What it covers |
|---|-----|----------------|
| 4 | `projects.md` | Overview + 2–3 featured project writeups (architecture, stack, lessons) |
| 5 | `how-i-work.md` | Engineering philosophy, collaboration style, code review approach |
| 6 | `tech-hot-takes.md` | Unpopular opinions on tools, languages, processes — with reasoning |
| 7 | `day-in-the-life.md` | What an ideal workday looks like, routines, deep work vs. meetings |
| 8 | `talks-writing.md` | Conference talks, blog posts, threads, any public writing |
| 9 | `open-source.md` | Repos contributed to, maintained, or notable PRs |

## Career Fit (1 doc)

| # | Doc | What it covers |
|---|-----|----------------|
| 10 | `opportunities.md` | What roles I'm looking for, industries, remote/onsite, team size, growth |

## Personal Color (5 docs)

| # | Doc | What it covers |
|---|-----|----------------|
| 11 | `travel.md` | Places visited, favorite trips, dream destinations |
| 12 | `hobbies.md` | What I do outside work (includes creative side projects) |
| 13 | `media.md` | Books, films, podcasts, games — recent and all-time favorites |
| 14 | `values.md` | What drives me, what I care about in work and life |
| 15 | `fun-facts.md` | Unexpected things — makes the bot delightful |

## File Structure

```
knowledge-base/
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

## Notes

- Each document will be chunked, embedded, and tagged with structured metadata (see [rag-vs-knowledge-graph.md](rag-vs-knowledge-graph.md) for the metadata-filtered RAG approach)
- Doc sizes should be 200–800 words each — enough for rich retrieval, not so large that chunks lose focus
- The 6/3/1/5 split keeps the balance professional-weighted while giving personal docs enough presence for the bot to feel human
