# Learning Notes — DDD/CQRS Study

> Personal learning journal for studying this repo, kept as markdown instead of paper.
> Start here, then jump to [tracker.md](tracker.md) for current status.

---

## How This Folder Is Organized

| File | Purpose |
|------|---------|
| [tracker.md](tracker.md) | **Start here.** Progress tracker, next action, session log |
| [flashcards.md](flashcards.md) | Self-quiz Q&A cards, organized by ADR |
| [mistakes-and-corrections.md](mistakes-and-corrections.md) | Every wrong answer I gave during drills + the correction + why |
| [bugs-found.md](bugs-found.md) | Real bugs found in this repo while studying — contribution candidates |
| [questions.md](questions.md) | Every question I ask, reframed, with the full-depth answer — the "extra 10 miles" file |
| [six-month-plan.md](six-month-plan.md) | Daily/monthly plan from finishing this repo through building a full career portfolio |
| `ADR-00X-*.md` | Deep-dive notes per architecture decision, with SA-system examples |

---

## ADR Notes

- [ADR-001: Cross-Aggregate Coordination](ADR-001-cross-aggregate-coordination.md)
- [ADR-002: Optimistic Concurrency Control](ADR-002-optimistic-concurrency-control.md)
- [ADR-003: Domain Events vs Integration Events](ADR-003-domain-vs-integration-events.md)
- [ADR-004: Outbox Pattern](ADR-004-outbox-pattern.md) *(drill still open)*
- ADR-005: Error Management System — not yet started
- ADR-006: Unobtrusive Integration Events — not yet started

---

## How to Use This Folder Day to Day

1. Open [tracker.md](tracker.md) — it always says what's next.
2. Read/finish the relevant ADR note.
3. Before moving on, self-quiz with [flashcards.md](flashcards.md) for that topic.
4. After any drill, log wrong answers in [mistakes-and-corrections.md](mistakes-and-corrections.md) — even (especially) the embarrassing ones. That's the highest-value file for review before interviews.
5. Update the session log table in [tracker.md](tracker.md) at the end of each session — one line, what I did, key takeaway.

---

## Why Markdown Instead of Paper

- Searchable, versioned, and pushed to my fork — never lost, never illegible
- Doubles as portfolio evidence: a public, timestamped record of deliberate study on a real DDD/CQRS codebase
- Cross-links between files (mistakes ↔ ADR notes ↔ flashcards) build a real knowledge graph, not isolated notes
