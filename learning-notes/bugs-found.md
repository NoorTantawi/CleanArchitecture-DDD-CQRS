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

## Bug #2 (candidate, unconfirmed): OutboxDbContext transaction enrollment

**Status:** Flagged for investigation, not yet confirmed as a bug
**Found during:** ADR-004 (Outbox Pattern) study

### Where
[`src/outbox/CleanArchitecture.Outbox/Persistence/EfCoreOutboxStore.cs`](../src/outbox/CleanArchitecture.Outbox/Persistence/EfCoreOutboxStore.cs)

### The concern
`EfCoreOutboxStore.AddAsync` calls `SaveChangesAsync()` on a **separate `OutboxDbContext`**, distinct from the main business `DbContext` that `TransactionCommandPipeline` manages. For ADR-004's core guarantee ("outbox write is atomic with business data") to hold true, both contexts need to share the same underlying database connection/transaction — this typically requires explicit transaction enrollment (e.g., sharing a `DbConnection` + `DbTransaction`, or using `TransactionScope`).

### Why it matters
If the two contexts are NOT sharing a transaction, there's a window where the business data commits but the outbox write fails (or vice versa) — reintroducing the exact dual-write problem the outbox pattern exists to solve.

### Next step
Read `TransactionCommandPipeline.cs` and `EfUnitOfWork.cs` together to confirm whether transaction sharing is actually wired up (e.g., via `DbContext.Database.UseTransaction(...)`), or whether this is a real gap. Do this as part of the pipeline deep-dive (tracker.md item #9).
