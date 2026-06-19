# TUI Commands

The terminal UI supports a command-driven interface alongside freeform RAG queries. This gives visitors two interaction paths: guided commands for quick scanning, and `ask` for deeper exploration.

## Design principle

A pure Q&A terminal puts cognitive load on the visitor — they need to know what to ask. Commands lower the barrier. A recruiter can run `resume`, `projects`, and `skills` and get everything they need in seconds, without typing a single natural language question.

## Command Set

### Navigation & Help

| Command | Behavior |
|---------|----------|
| `help` | Lists all available commands with brief descriptions |
| `clear` | Clears the terminal screen |
| `whoami` | Displays meta info — "You're talking to {name}'s portfolio bot. Ask me anything or try a command." |

### Resume Essentials

| Command | Behavior |
|---------|----------|
| `about` | Quick bio — who they are, current role, what they're looking for. **[TBD: LLM-summarized from experience.md + opportunities.md + other docs]** |
| `resume` | Displays full resume inline in the terminal with formatting |
| `resume download` | Triggers PDF download of the resume |
| `skills` | Lists technical skills grouped by category (languages, frameworks, tools) |
| `skills <category>` | Filters skills by category, e.g., `skills languages` |

### Deep Professional

| Command | Behavior |
|---------|----------|
| `experience` | Work history summary — companies, roles, dates |
| `experience <n>` | Detail view of the nth experience entry |
| `projects` | Lists all projects with one-line descriptions |
| `projects <name>` | Full writeup for a specific project (architecture, stack, lessons) |
| `opensource` | Open source contributions, repos, notable PRs |
| `talks` | Conference talks, blog posts, publications |
| `how-i-work` | Engineering philosophy, collaboration style, code review approach |
| `hot-takes` | Tech hot takes with reasoning |
| `day-in-life` | What a typical workday looks like |

### Career Fit

| Command | Behavior |
|---------|----------|
| `opportunities` | What roles they're looking for, industries, remote/onsite, team size |

### Personal Color

| Command | Behavior |
|---------|----------|
| `travel` | Places visited, favorite trips, dream destinations |
| `hobbies` | Interests and activities outside work |
| `media` | Books, films, podcasts, games — favorites |
| `values` | Personal values and what drives them |
| `fun-facts` | Unexpected or quirky facts about them |

### Contact

| Command | Behavior |
|---------|----------|
| `contact` | Email, LinkedIn, GitHub, preferred contact method |

### Feedback

| Command | Behavior |
|---------|----------|
| `feedback <message>` | Submits feedback about the portfolio experience |

Visitors can leave thoughts, report issues, or suggest improvements. Feedback is stored as a JSONL file — no database needed at this scale.

```
# Terminal interaction
> feedback The resume download didn't work on mobile
Thanks for your feedback! It'll help improve this portfolio.
```

```json
// logs/feedback-2026-06-19.jsonl
{"timestamp": "2026-06-19T14:35:00Z", "message": "The resume download didn't work on mobile", "ip_hash": "a1b2c3..."}
```

### Freeform RAG

| Command | Behavior |
|---------|----------|
| `ask <question>` | Freeform natural language query. The RAG pipeline retrieves relevant chunks and generates an answer. |

## Landing experience

When a visitor first opens the site, the terminal auto-types a welcome message:

```
Welcome to {name}'s portfolio terminal.
Type 'help' to see available commands, or 'ask <question>' to learn more about me.
Try: ask what projects has {name} built with Go?
```

This immediately teaches the visitor how to interact.

## Command vs. RAG balance

| Visitor type | Likely path |
|-------------|-------------|
| **Quick scanner** | `resume` → `resume download` → done |
| **Technical deep-dive** | `projects` → `projects <name>` → `ask what was the hardest part?` |
| **Culture fit check** | `how-i-work` → `values` → `day-in-life` |
| **Curious explorer** | `help` → tries several commands → `ask random question` |
| **Recruiter with specific question** | `ask does this candidate have experience with distributed systems?` |

## Implementation note

Commands that map 1:1 to a knowledge base document (e.g., `travel` → `travel.md`) can either:
- **Direct fetch:** Retrieve and display the document directly (fastest, simplest)
- **Summarize via LLM:** Generate a condensed version (adds cost but feels more conversational)

For v1, direct fetch for document-mapped commands is recommended. `ask` is the only command that hits the full RAG pipeline. `feedback` writes to a JSONL file — no database, no LLM.
