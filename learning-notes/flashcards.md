# Flashcards

> Self-quiz format. Cover the answer, read the question, say the answer out loud, then check.
> Organized by ADR. New cards get appended as new topics are covered.

---

## ADR-001: Cross-Aggregate Coordination

**Q: What is an aggregate?**
> A cluster of objects treated as one consistency unit. It protects its own business rules. Outsiders interact only through the root.

**Q: What is an aggregate root?**
> The single entry point into an aggregate. Outsiders never touch inner objects (e.g. InvoiceLine) directly — only through the root (Invoice).

**Q: What is a bounded context?**
> A business area with its own language and model. The same word ("Account") can mean different things across contexts (Sales vs Accounting vs Auth).

**Q: How should a domain event be named?**
> Past tense — it represents something that already happened. `InvoicePostedEvent`, not `InvoiceUpdatedEvent` (too vague) or "post invoice event" (wrong tense).

**Q: How many repositories should a command handler inject?**
> One — from its own bounded context only. Never inject repositories from other contexts; that's Option 1 (rejected pattern).

**Q: Why was "Mediator.Send(Command) from Handler" rejected as a coordination pattern?**
> Commands are for external requests, not internal coordination. It creates hidden command chains and unclear transaction boundaries, and violates bounded context autonomy.

**Q: When is a Domain Service the right tool (not a command handler)?**
> When coordination logic is complex, spans multiple aggregates, and no single aggregate owns the rule — e.g., checking credit limit + stock + contract validity all together before confirming a sale.

**Q: What actually rolls back when a domain event handler throws an exception?**
> The database transaction — specifically, `SaveChangesAsync()` is never reached, so nothing was ever written. NOT "the event" — events have no database presence to undo.

**Q: Is event processing in `DomainEventsPipeline` stack-based or batch-based?**
> Batch-based (breadth-first). Batch 1 (all currently queued events) fully completes before Batch 2 (events raised during Batch 1) begins.

**Q: What architecture test enforces ADR-001?**
> `CommandHandlers_Should_Use_Repository_From_Same_BoundedContext` — fails the build if a handler injects a repo from another bounded context.

---

## ADR-002: Optimistic Concurrency Control

**Q: What HTTP status code does a concurrency conflict return?**
> 409 Conflict. (Not 400 — the request was valid. Not 407 — unrelated to proxy auth.)

**Q: What SQL actually happens when EF saves an entity with a RowVersion concurrency token?**
> `UPDATE ... WHERE Id = @id AND RowVersion = @originalRowVersion`. If 0 rows are affected, EF throws `DbUpdateConcurrencyException`.

**Q: What two EF Core configuration calls are needed for RowVersion to work as a concurrency token?**
> `.IsRowVersion()` (maps to SQL Server ROWVERSION type, auto-incremented by DB) AND `.IsConcurrencyToken()` (includes it in the WHERE clause). Both required.

**Q: Why was pessimistic locking rejected for this system?**
> Poor performance under load, deadlock risk, and poor UX for web apps (a user who steps away holds a lock indefinitely). Optimistic fits "conflicts are rare" reality better.

**Q: Does optimistic concurrency protect a business rule that spans TWO different aggregate rows (e.g., a budget shared across many payment records)?**
> No — not by default. It only detects conflicts on the SAME row/aggregate. A rule spanning multiple different rows needs a dedicated owning aggregate (e.g., a SeasonBudget aggregate) that every write must go through.

**Q: Should a failed financial operation (e.g., posting an invoice) auto-retry after a 409, or force the user to refresh?**
> Force the user to refresh/intervene. The user made a decision based on stale data — auto-retrying could apply their old numbers to a now-different state, causing silent business-rule violations.

---

## ADR-003: Domain Events vs Integration Events

**Q: What's the core difference between `IDomainEventHandler` and `IIntegrationEventHandler`?**
> `IDomainEventHandler` runs INSIDE the transaction — failure rolls everything back. `IIntegrationEventHandler` runs AFTER commit via the outbox — failure retries, does not roll back the core operation.

**Q: Can the same domain event have both types of handlers?**
> Yes. E.g. `WorkOrderCompletedEvent` has `IDomainEventHandler`s for Asset/Technician state changes AND an `IIntegrationEventHandler` for the completion email.

**Q: Decision flowchart — how do you pick which interface a new handler should use?**
> Does it modify aggregate state? Yes → IDomainEventHandler. No → does failure affect business correctness? Yes → IDomainEventHandler anyway. No → IIntegrationEventHandler.

**Q: Why was the "single interface + attribute" design (Option 4) rejected?**
> No compile-time safety — a developer could forget the `[Transactional]` attribute and the bug would only surface at runtime, possibly in production with money involved.

**Q: A handler calls a government tax API that must eventually succeed but can retry. What handler type, and what extra design element is required?**
> `IIntegrationEventHandler`, with an **idempotency key** (e.g. the invoice ID) included in every API call so retries don't cause duplicate submissions.

**Q: Can one handler both update an aggregate's state AND call an external API?**
> Technically yes, but it's wrong design — it mixes two different consistency guarantees into one handler. Split into two handlers: one `IDomainEventHandler` for the state change, one `IIntegrationEventHandler` for the external call.

---

## ADR-004: Outbox Pattern

**Q: What is the "dual-write problem"?**
> When a service must update a database AND publish an event/call an external system, but cannot do both atomically — one can succeed while the other fails.

**Q: How does the outbox pattern turn "dual-write" into "single-write"?**
> By writing the event as a row in an outbox table, inside the SAME database transaction as the business data. If the transaction commits, the event is guaranteed to exist; if it rolls back, so does the event row.

**Q: What delivery guarantee does the outbox provide — exactly-once or at-least-once?**
> At-least-once. An event can be processed more than once (e.g., if the app crashes after the handler runs but before `MarkAsProcessedAsync`). Every integration handler MUST be idempotent.

**Q: What do `UPDLOCK`, `ROWLOCK`, and `READPAST` do in the outbox query, and why do they matter together?**
> UPDLOCK locks the row for this worker; ROWLOCK keeps the lock scoped to just that row; READPAST makes other workers skip locked rows instead of waiting. Together they let multiple `OutboxProcessor` instances run in parallel safely.

**Q: What happens when a message's RetryCount reaches MaxRetries?**
> It's moved to the Dead Letter table — no more automatic retries. Requires human investigation, and can be manually replayed after the underlying issue is fixed.

**Q: Why does an idempotency check belong in the handler and not in the outbox infrastructure?**
> The outbox guarantees delivery, not exactly-once execution. Only the handler knows what "already done" means for its specific side effect (e.g., "was this exact email already sent for this invoice ID?").

---

## Quick-Fire Round (mixed review)

**Q: Aggregate vs Aggregate Root — what's the difference?**
> Aggregate = the whole cluster/consistency unit. Aggregate Root = the one object within it that's the entry point for outside access.

**Q: Command handler touches how many bounded contexts?**
> One. Coordination with other contexts happens via events, not direct repository access.

**Q: What's the anchor for "Integration Event Handler"?**
> A letter sent after the internal meeting is done and filed — if the post office loses it, you resend it; you don't un-file your internal records.

**Q: What's the anchor for "Optimistic Concurrency"?**
> Booking the last flight seat online — nobody's blocked from looking, but whoever completes payment first wins; the second person is told to refresh.

**Q: True or false — "the event rolled back" is a correct way to describe a failed cross-aggregate operation.**
> False. Events don't roll back (no DB presence). The transaction never commits — that's the accurate description.
