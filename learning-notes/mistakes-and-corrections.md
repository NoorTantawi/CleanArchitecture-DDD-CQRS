# Mistakes and Corrections Log

> Every wrong or imprecise answer I gave during drills, and the correction.
> This is the highest-value file for review — mistakes stick better than clean explanations.

---

## ADR-001: Cross-Aggregate Coordination

### Mistake 1 — Command handler injecting multiple repositories
**What I said (Drill 1):** Listed `sale, JournalEntry, Farmer payment, buyer account, store invoice` as repositories the `ConfirmSaleOrder` command handler should inject.

**Why it's wrong:** This is exactly the anti-pattern ADR-001 rejects (Option 1). The command handler should inject **only** `IRepository<SaleOrder, Guid>`. Everything else is updated by domain event handlers reacting to `SaleOrderConfirmedEvent`, each in their own bounded context.

**Correction:** One command handler = one aggregate = one repository. Cross-context updates happen via events, never via direct injection.

---

### Mistake 2 — Aggregate/event naming
**What I said:** Named the aggregate "ConfirmSaleOrder" and the event "confirm sale event."

**Why it's wrong:** "Confirm" is an *operation*, not a *thing*. The aggregate is `SaleOrder`. Domain events must be named in **past tense** because they represent something that already happened — `SaleOrderConfirmedEvent`, not "confirm sale event."

**Correction:** Aggregate = noun (`SaleOrder`). Event = past-tense verb phrase (`SaleOrderConfirmedEvent`).

---

### Mistake 3 — Misunderstanding what "orphaned data" means for async handlers
**What I said:** Claimed that if handler failures don't roll back (point 7 / async handlers), we'd get orphaned data.

**Why it's wrong:** Async/integration handlers (email, notifications) run **after** the transaction already committed. The core data (SaleOrder, JournalEntry, stock) was already safely committed by the transactional handlers before the async ones even run. A failed email doesn't orphan anything — the sale is still valid and complete.

**Correction:** Orphaned data would only happen if a *transactional* write (something the business depends on) was moved into an async/outbox handler by mistake. Non-critical side effects (email, audit) are safe to run post-commit.

---

### Mistake 4 — Domain Service definition
**What I said:** Described a domain service as being for "master data, where we write to only one table" (e.g., produce type, container type).

**Why it's wrong:** Writing to one table with no coordination is just a normal command handler — nothing "domain service" about it. A domain service exists for **complex coordination logic that spans multiple aggregates and doesn't naturally belong to any single one of them.**

**Correction:** Domain service example — before confirming a large sale: check buyer credit limit (BuyerAccount) + stock availability (Inventory) + active contract (FarmerContract), all together, as one rule. No single aggregate owns "can this sale proceed?" — that's a domain service's job.

---

### Mistake 5 — "The event rolls back"
**What I said (checkpoint question):** "The sale order remains as pending confirmation, and that confirmed event rolls back as journal entry aggregate throws a domain exception."

**Why it's wrong:** Events are in-memory notifications. They have no database presence and cannot "roll back." What actually rolls back is the **database transaction** — specifically, `SaveChangesAsync()` is never reached because the exception propagates up before it's called. `TransactionCommandPipeline`'s catch block then calls `RollbackAsync()` as a safety net, but technically nothing was ever written to the database in the first place.

**Correction:** Say "the transaction never commits, so nothing persists" — not "the event rolls back." Events don't roll back; database writes do (or rather, don't happen at all).

---

### Mistake 6 — Batch processing described as a stack
**What I said:** Described chained domain events (Event A raises Event B) as stack-based — Event B "interrupts" Event A and processes first, "top of the stack."

**Why it's wrong:** `DomainEventsPipeline.ProcessDomainEventsInBatches` is **breadth-first (batch-by-batch)**, not depth-first (stack-based). Batch 1 (all currently queued events) runs to full completion before Batch 2 (events raised during Batch 1) begins.

**Correction:** Picture a queue of batches, not a stack of interruptions. If two handlers in Batch 1 each raise a new event, both new events land in Batch 2 together — neither interrupts the other.

---

### Mistake 7 — Calling the command handler a "domain service"
**What I said (revision round):** "CompleteWorkOrderCommandHandler ... because it's a domain service to handle the complete work order command."

**Why it's wrong:** `CompleteWorkOrderCommandHandler` is an **application command handler**, not a domain service. These are distinct patterns: a command handler receives one command and touches one aggregate; a domain service coordinates complex logic across multiple aggregates (see Mistake 4).

**Correction:** Command handler ≠ domain service. Don't reach for the domain service label just because a handler avoids touching other aggregates directly — that's just it following ADR-001 correctly.

---

### Mistake 8 — Why async handlers don't roll back
**What I said (revision round):** "We don't roll back anything as they are in memory" (referring to integration/async events).

**Why it's wrong:** Integration events are NOT just in memory — they are written to the **outbox table in the database**, inside the same transaction as everything else (`DomainEventsPipeline.cs:101`, `WriteIntegrationEventsToOutbox`). The real reason they don't roll back the business operation is that they run **after the transaction has already committed**, in a separate background process, at a later point in time. A failed email cannot reach back in time to undo a committed transaction.

**Correction:** The distinction is about **timing** (before vs. after commit), not about **storage** (memory vs. database). Integration events are durably stored — that's the entire point of the outbox pattern (see ADR-004).

---

## ADR-002: Optimistic Concurrency Control

### Mistake 9 — Wrong HTTP status code
**What I said (Drill 2):** "It shows error 407."

**Why it's wrong:** HTTP 407 is "Proxy Authentication Required" — unrelated to concurrency. The correct code is **HTTP 409 Conflict**, confirmed in `ExceptionHandlingMiddleware.cs:109`.

**Correction:** Memorize: **409 = valid request, but the resource's current state conflicts with what you expected.** 400 = malformed request. 407 = proxy auth. Don't mix these up — they signal completely different problems to API clients.

---

### Mistake 10 — Where optimistic concurrency "is not enough"
**What I said:** Used "two users pay the same farmer" as the example where optimistic concurrency fails.

**Why it's wrong:** That scenario is actually **protected fine** by optimistic concurrency — both users read the same `FarmerPayment` row with the same RowVersion; the second save gets a 409. Same-aggregate conflicts are exactly what RowVersion is built for.

**Correction:** The real gap is **cross-aggregate invariants** — a business rule that spans two *different* aggregate rows, where neither write conflicts with the other because they're on different records. Example: two invoices posted simultaneously, both checking `CustomerAccount` credit limit — if the check reads a value but doesn't also write to `CustomerAccount` in the same transaction, both can pass the check and together violate the limit. Fix: make sure the aggregate that OWNS the invariant is always read AND written, so its RowVersion catches the conflict.

---

## ADR-003: Domain Events vs Integration Events

### Mistake 11 — Confusing repository interfaces with handler-type interfaces
**What I said (Drill 3):** Answered "IFarmerAccount interface, IWhatsappNotification, IGovernmentTaxLiability" when asked which **handler interface** (`IDomainEventHandler` vs `IIntegrationEventHandler`) applies to each scenario.

**Why it's wrong:** Those are the **dependency interfaces you'd inject into a handler** (repositories/services) — a different concept from the **handler category interface** that determines when/how the handler runs.

**Correction:**
```csharp
public class FarmerPaymentObligationHandler
    : IDomainEventHandler<SaleOrderConfirmedEvent>   // ← THIS is "which interface"
{
    private readonly IFarmerAccountRepository _repo;  // ← THIS is an injected dependency
    ...
}
```
When asked "which interface," the answer is always `IDomainEventHandler` or `IIntegrationEventHandler` — never the name of an injected service or repository.

---

## Pattern Across All Mistakes

Looking back across all 11 corrections, three recurring blind spots:

1. **Mixing up "what a thing is" vs "what a thing depends on."** (Mistakes 7, 11) — a command handler isn't a domain service just because it's simple; a handler's *type* isn't the same as its *dependencies*.
2. **Reaching for the wrong mental model under pressure.** (Mistakes 5, 6, 9) — "the event rolls back" instead of "the transaction never commits"; "stack" instead of "batch queue"; guessing a status code instead of recalling the actual one.
3. **Same-aggregate vs cross-aggregate confusion.** (Mistakes 3, 10) — this is the single most repeated gap. Before answering any "is X protected?" question, first ask: **are we talking about ONE aggregate or TWO DIFFERENT aggregates?** Most of these mistakes disappear once that question is asked first.
