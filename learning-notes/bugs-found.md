# Bugs Found While Studying

> Real issues discovered by reading this repo carefully during ADR study.
> These are candidate contributions back to the upstream project — the "Phase 0 graduation deliverable."

---

## Bug #1: Concurrency conflict error always blames "Asset"

**Status:** Not yet fixed / not yet contributed
**Found during:** ADR-002 (Optimistic Concurrency Control) study
**Severity:** Low-medium — misleading error message, not a data integrity issue

### Where
[`src/CleanArchitecture.Cmms.Api/Middlewares/ExceptionHandlingMiddleware.cs:101-111`](../src/CleanArchitecture.Cmms.Api/Middlewares/ExceptionHandlingMiddleware.cs)

### The code
```csharp
private async Task WriteConcurrencyResponseAsync(HttpContext context, DbUpdateConcurrencyException ex)
{
    _logger.LogWarning(ex, "Concurrency conflict occurred.");

    var result = Result.Failure(AssetErrors.ConcurrencyConflict);   // ← always Asset, regardless of entity

    context.Response.ContentType = "application/json";
    context.Response.StatusCode = (int)HttpStatusCode.Conflict;

    await context.Response.WriteAsync(JsonSerializer.Serialize(result, _jsonOptions));
}
```

### The problem
Every aggregate root in this system (`WorkOrder`, `Asset`, `Technician`) inherits `RowVersion` from the base `AggregateRoot<TId>` class, so any of them can throw `DbUpdateConcurrencyException`. But the middleware's catch handler hardcodes `AssetErrors.ConcurrencyConflict`, whose message reads:

> "Asset was modified by another user. Please refresh and try again."

If a `WorkOrder` or `Technician` conflict occurs, the client still receives this Asset-specific message — actively misleading whoever is debugging the conflict or the end user reading the error.

### How to reproduce
1. Two clients load the same `Technician` (or `WorkOrder`) simultaneously.
2. Both attempt an update (e.g., both try to assign the same technician to different work orders).
3. The second request throws `DbUpdateConcurrencyException`.
4. Response body says "Asset was modified by another user" — even though no Asset was involved.

### Proposed fix (two options)

**Option A — generic entity-agnostic error (simplest):**
```csharp
// A new shared error not tied to any specific domain
public static readonly Error ConcurrencyConflict = Error.Conflict(
    "Concurrency.Conflict",
    "This record was modified by another user. Please refresh and try again.");
```

**Option B — inspect which entity actually conflicted (more precise):**
```csharp
private async Task WriteConcurrencyResponseAsync(HttpContext context, DbUpdateConcurrencyException ex)
{
    var entityName = ex.Entries.FirstOrDefault()?.Entity.GetType().Name ?? "Record";
    var result = Result.Failure(Error.Conflict(
        "Concurrency.Conflict",
        $"{entityName} was modified by another user. Please refresh and try again."));
    // ...
}
```

Option A is the safer, smaller PR. Option B is more informative but touches more of the error catalogue pattern (see ADR-005 once that's studied — may be worth revisiting this fix after ADR-005 to align with the broader error system).

### Next step
Revisit after ADR-005 (Error Management System) is studied, since the fix should follow whatever error cataloguing convention that ADR establishes. Then open a PR against `NoorTantawi/CleanArchitecture-DDD-CQRS` (this fork) — or upstream if this fork tracks an original repo by Mohammad Shakhtour.

---

## Bug #2 — CONFIRMED: OutboxDbContext does not share the business transaction

**Status:** Confirmed by reading `EfUnitOfWork.cs`, `EfTransaction.cs`, `OutboxDbContext.cs`, and `ServiceCollectionExtensions.cs` (outbox project) — 2026-07-18
**Found during:** ADR-004 (Outbox Pattern) study, follow-up deep dive
**Severity:** High — undermines ADR-004's core guarantee

### Where
- [`src/core/CleanArchitecture.Core.Infrastructure.Persistence/EfCore/EfUnitOfWork.cs`](../src/core/CleanArchitecture.Core.Infrastructure.Persistence/EfCore/EfUnitOfWork.cs)
- [`src/outbox/CleanArchitecture.Outbox/Persistence/EfCoreOutboxStore.cs`](../src/outbox/CleanArchitecture.Outbox/Persistence/EfCoreOutboxStore.cs)
- [`src/outbox/CleanArchitecture.Outbox/ServiceCollectionExtensions.cs`](../src/outbox/CleanArchitecture.Outbox/ServiceCollectionExtensions.cs)

### The evidence

`EfUnitOfWork.BeginTransactionAsync` opens a transaction on the main business `DbContext` only:
```csharp
public async Task<ITransaction> BeginTransactionAsync(CancellationToken ct = default)
    => new EfTransaction(await _db.Database.BeginTransactionAsync(ct));
```

`OutboxDbContext` is registered as a completely independent context, with its own connection string, no transaction sharing:
```csharp
// ServiceCollectionExtensions.AddOutbox
services.AddDbContext<OutboxDbContext>(dbOptions =>
    dbOptions.UseSqlServer(connectionString, ...));
```

There is no call anywhere to `OutboxDbContext.Database.UseTransaction(...)`, no shared `DbConnection`, no `TransactionScope` wrapping both contexts.

### Why it matters
`EfCoreOutboxStore.AddAsync` calls `_context.SaveChangesAsync()` on `OutboxDbContext` **while still inside** `DomainEventsPipeline.Handle` — which runs *before* the main `TransactionCommandPipeline` calls its own `SaveChangesAsync()`/`CommitAsync()`. Because `OutboxDbContext` has no enlisted transaction, that `SaveChangesAsync()` commits **immediately and independently**.

Concrete failure: if the main business transaction later fails (e.g., a `DbUpdateConcurrencyException` on `SaveChangesAsync`) and rolls back, the `OutboxMessage` row **survives anyway** — it already committed in its own separate transaction moments earlier. Result: an outbox row exists claiming an event happened (e.g. `WorkOrderCompletedEvent`), the background processor eventually sends an email/notification for it — but the underlying business change was never actually persisted. This reintroduces the exact dual-write problem the outbox pattern exists to solve.

### Proposed fix
Share the connection/transaction between the two contexts, e.g.:
```csharp
// share the same open DbConnection + DbTransaction across both contexts
outboxDbContext.Database.SetDbConnection(mainDbContext.Database.GetDbConnection());
await outboxDbContext.Database.UseTransactionAsync(
    mainDbContext.Database.CurrentTransaction!.GetDbTransaction());
```
Or restructure so the outbox table is a `DbSet` on the *main* business context instead of a separate `OutboxDbContext`, removing the split entirely for the single-database deployment case (the separate-context design exists to support a future separate outbox database — see ADR-004's "Why Separate?" rationale — so this fix needs to preserve that future option, not just delete it).

### Next step
Checked `TransactionHandling/TransactionCommitTests.cs` — it only covers the background *processor's* per-message transaction (marking `ProcessedAt`), not write-time atomicity between the outbox insert and the business transaction. It does not test or disprove this finding. No existing test covers the scenario described above. Worth writing one (business transaction rolls back → assert outbox row does NOT exist) as part of the pipeline deep-dive (tracker.md item #9), before deciding whether to open a PR.
