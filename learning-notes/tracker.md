# Learning Tracker

> Single source of truth for where I am in the DDD/CQRS repo study, and what's next.
> Update this file every session — it replaces the notebook.

---

## Right Now

**Next action:** Start ADR-005: Error Management System

---

## Phase 0: DDD/CQRS Repo Study

| # | Topic | Status | Notes File | Drill Status |
|---|-------|--------|-----------|---------------|
| 1 | ADR-001: Cross-Aggregate Coordination | ✅ Done | [ADR-001](ADR-001-cross-aggregate-coordination.md) | Drill 1 + revision round: done |
| 2 | ADR-002: Optimistic Concurrency Control | ✅ Done | [ADR-002](ADR-002-optimistic-concurrency-control.md) | Drill 2: done |
| 3 | ADR-003: Domain Events vs Integration Events | ✅ Done | [ADR-003](ADR-003-domain-vs-integration-events.md) | Drill 3: done |
| 4 | ADR-004: Outbox Pattern | ✅ Done | [ADR-004](ADR-004-outbox-pattern.md) | Drill 4: done — found retry-exhaustion mistake + confirmed the transaction bug |
| 5 | ADR-005: Error Management System | ⬜ **Next up** | — | — |
| 6 | ADR-006: Unobtrusive Integration Events | ⬜ Not started | — | — |
| 7 | CQRS Read Side (Dapper, read repositories) | ⬜ Not started | — | — |
| 8 | Custom Mediator internals | ⬜ Not started | — | — |
| 9 | All 4 Pipelines (logging, validation, transaction, domain events) | 🟡 Partial (transaction + domain events covered via ADR-001/002) | — | — |
| 10 | Architecture Tests | ⬜ Not started | — | — |
| 11 | Integration Tests (Testcontainers) | ⬜ Not started | — | — |

**Phase 0 graduation deliverable:** submit a PR fixing a bug in [bugs-found.md](bugs-found.md) — two candidates now: the `AssetErrors.ConcurrencyConflict` hardcoding, or the confirmed `OutboxDbContext` transaction-sharing gap (Bug #2, higher severity, real correctness issue).

---

## Phase 0 Estimated Completion

~2-3 weeks at current pace (topics 5–11 remaining).

---

## Six-Month Plan (starts after Phase 0)

See [six-month-plan.md](six-month-plan.md) for the full month-by-month plan (SA system hardening → testing/observability → Docker/CI → K8s/RabbitMQ → AI engineering → security/system design).

| Month | Theme | Status |
|-------|-------|--------|
| 1 | Apply DDD/CQRS/ADRs to SA system | ⬜ Not started |
| 2 | Testing + Observability | ⬜ Not started |
| 3 | Docker + CI/CD | ⬜ Not started |
| 4 | Kubernetes + RabbitMQ | ⬜ Not started |
| 5 | AI Engineering (Claude API, MCP, evals) | ⬜ Not started |
| 6 | Security + System Design + Portfolio | ⬜ Not started |

---

## Study Aids

- [flashcards.md](flashcards.md) — self-quiz cards, organized by ADR
- [mistakes-and-corrections.md](mistakes-and-corrections.md) — every wrong/imprecise answer and its correction, with reasoning
- [bugs-found.md](bugs-found.md) — real bugs discovered in this repo while studying, candidates for contribution
- [questions.md](questions.md) — every question I've asked, reframed, with the full-depth answer

---

## Session Log

| Date | What I did | Key takeaway |
|------|-----------|--------------|
| 2026-05-31 | Started repo study, ADR-001 intro | Aggregates protect invariants; command handlers touch one aggregate |
| 2026-06-01 | ADR-001 concepts deep dive | Bounded context, domain event, transactional vs integration handler |
| 2026-06-03 | ADR-001 Drill 1 (PostInvoice design) | Command handler injects ONE repo only; domain service ≠ command handler |
| 2026-06-13 | ADR-001 revision + checkpoint questions | Transaction rolls back, not the event; batch processing is breadth-first, not stack-based |
| 2026-06-15 | ADR-002 intro + Drill 2 | RowVersion mechanism, HTTP 409 (not 407!), optimistic concurrency protects ONE aggregate at a time |
| 2026-06-28 | ADR-002 detailed notes, ADR-003 taught + Drill 3 | Domain vs integration event handlers; idempotency keys for external APIs; found the AssetErrors bug |
| 2026-07-11 | Career discipline discussion, 6-month plan designed | Evidence > credentials; SA system is the lab, not a side project |
| 2026-07-18 | ADR-004 taught (outbox pattern); notes folder organized and pushed to fork | Dual-write problem, at-least-once delivery, idempotent handlers required |
| 2026-07-18 | Deep dive on outbox dispatch internals (questions.md started); Drill 4 answered | Confirmed the OutboxDbContext transaction-sharing bug is real, not just theoretical; retry exhaustion is count-based with no delay between attempts, not time-based |

*(Add a line every session — this becomes your interview story bank.)*
