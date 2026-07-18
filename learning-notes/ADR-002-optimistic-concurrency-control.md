# ADR-002: Optimistic Concurrency Control

> Personal learning notes — Noor Tantawi  
> Source repo: CleanArchitecture-DDD-CQRS (open source CMMS)

---

## The Problem: Lost Updates and Race Conditions

### What is a Lost Update?

A lost update happens when two users read the same record, both make changes based on what they read, and the second writer silently overwrites the first writer's changes — with no error, no warning, no trace.

**Real-life anchor:** Two people editing a shared printed document. Person A crosses out "100 units" and writes "70 units." Person B crosses out "100 units" and writes "80 units." They both hand the document back. Whoever hands it back last wins. The other person's work is erased as if it never happened.

**Technical sequence (without any concurrency control):**

```
Time 1:  User A reads Invoice #1042  (Total = 500, Status = Draft)
Time 2:  User B reads Invoice #1042  (Total = 500, Status = Draft)
Time 3:  User A adds a line item     (Total = 700)
Time 4:  User A saves                (DB: Total = 700)
Time 5:  User B posts the invoice    (based on Total = 500, which is now stale)
Time 6:  User B saves                (DB: Status = Posted, Total = 500)

Result: User A's line item silently disappeared.
        The posted invoice is missing data.
```

This is the lost update problem. It is silent, insidious, and destroys financial data integrity.

---

### What is a Race Condition?

A race condition is when the correctness of an outcome depends on the timing and order of concurrent operations. The system behaves differently depending on who "wins the race."

**SA Example:**

```
Time 1:  User A reads CustomerAccount (Balance = 8,000, CreditLimit = 10,000)
Time 2:  User B reads CustomerAccount (Balance = 8,000, CreditLimit = 10,000)
Time 3:  User A checks: 8,000 + 1,500 = 9,500 < 10,000 → PASS
Time 4:  User B checks: 8,000 + 1,800 = 9,800 < 10,000 → PASS
Time 5:  User A commits   (Balance = 9,500)
Time 6:  User B commits   (Balance = 11,300) ← credit limit violated
```

Both users passed the check. Neither saw the other. The constraint was violated because the check and the write are not atomic.

---

## Core Definitions (Precise)

### Concurrency Control
A set of mechanisms that ensure concurrent operations produce correct results — as if they ran one at a time (serially).

There are two families: **Pessimistic** and **Optimistic**.

---

### Pessimistic Concurrency Control
Lock the data before you read it. No other process can read or write the locked rows until you release the lock.

**Mechanism in SQL Server:** `SELECT ... WITH (UPDLOCK, HOLDLOCK)`

```sql
BEGIN TRANSACTION
  SELECT * FROM Invoices WITH (UPDLOCK, HOLDLOCK) WHERE Id = @id
  -- row is now locked, nobody else can touch it
  UPDATE Invoices SET ...
COMMIT
-- lock released
```

**Analogy:** A changing room with a lock. One person goes in, locks the door. Everyone else waits outside — they cannot even look. When the person comes out, the next one enters.

**When correct:** When conflicts are frequent, when the cost of a conflict is very high (e.g., exclusive resource reservation), or when the window between read and write is very short (in-memory, same millisecond).

**Why rejected in web applications:**
- A user opens a form → row is locked
- User gets a phone call → lock held for 20 minutes
- Entire system stalls waiting for that one lock
- Under load: deadlocks form when Process A holds Lock 1 waiting for Lock 2, and Process B holds Lock 2 waiting for Lock 1 — both wait forever

---

### Optimistic Concurrency Control
Do not lock. Read freely. At the moment of saving, **check whether anyone else changed the data since you read it.** If yes, reject the write and tell the user to refresh.

**Core assumption:** Conflicts are rare. Most of the time, users edit different records. Locking everyone for the occasional conflict is not worth the performance cost.

**Analogy:** Booking the last flight seat online. Two people see the seat as available. Neither is blocked from viewing it. Whoever completes payment first gets the seat. The second person sees "this seat is no longer available — please select another."

---

### RowVersion (SQL Server)

A special database column type (`ROWVERSION` / `TIMESTAMP` in SQL Server) that:
- Is **automatically managed by the database** — you never set it manually
- Changes to a new unique value every time the row is updated
- Is an 8-byte binary value (`byte[]` in C#)
- Is guaranteed unique within the database at any point in time

**SQL Server behavior:**
```sql
-- When you INSERT a row:
INSERT INTO WorkOrders (Id, Title, ...) VALUES (...)
-- RowVersion is automatically set to e.g. 0x0000000000000123

-- When you UPDATE the row:
UPDATE WorkOrders SET Status = 'Completed' WHERE Id = @id
-- RowVersion automatically becomes 0x0000000000000124
-- You did nothing — the database did this
```

---

### Concurrency Token
A property on an entity that EF Core includes in the `WHERE` clause of `UPDATE` and `DELETE` statements. If the value in the database no longer matches the value EF loaded, EF knows the row was changed by someone else and throws `DbUpdateConcurrencyException`.

`RowVersion` is used as a concurrency token.

---

### DbUpdateConcurrencyException
The exception EF Core throws when a `SaveChangesAsync()` call affects zero rows because the concurrency token no longer matches.

This is not an application bug. It is the intended behavior of optimistic concurrency. You catch it and tell the user to refresh.

---

### HTTP 409 Conflict
The HTTP status code returned to the client when a concurrency conflict occurs.

**Precise meaning:** "The request was valid and well-formed, but the current state of the resource on the server conflicts with what the client expected."

It is NOT:
- `400 Bad Request` — the request itself was not malformed
- `500 Internal Server Error` — this is not an unexpected system failure
- `407 Proxy Authentication Required` — completely unrelated

---

### Optimistic vs Pessimistic — Decision Table

| Factor | Optimistic | Pessimistic |
|--------|-----------|-------------|
| Conflict frequency | Low (rare) | High (frequent) |
| Lock duration | No lock held | Lock held during read→write |
| Performance under low conflict | Excellent | Wasted overhead |
| Performance under high conflict | Many retries | Predictable serialization |
| Deadlock risk | None | High if multiple locks |
| User experience | Occasional "refresh" message | Waiting/timeouts |
| Web application suitability | Excellent | Poor |
| Short critical sections (ms) | Fine | Also fine |

---

## How This Repo Implements It — Three Layers

### Layer 1 — Domain: AggregateRoot

File: `src/core/CleanArchitecture.Core.Domain/Abstractions/AggregateRoot.cs`

```csharp
public abstract class AggregateRoot<TId> : AuditableEntity<TId>, IAggregateRoot
{
    [Timestamp]
    public byte[] RowVersion { get; protected set; } = default!;

    protected AggregateRoot() { }
    protected AggregateRoot(TId id) : base(id) { }
}
```

Every aggregate root inherits `RowVersion` automatically.  
`WorkOrder`, `Asset`, `Technician` all have it — without a single line of code in each.

**`[Timestamp]` attribute** = tells EF Core this property maps to a SQL Server `ROWVERSION` column that is database-managed and should be treated as a concurrency token.

---

### Layer 2 — Infrastructure: EF Configuration

```csharp
builder.Property(e => e.RowVersion)
    .IsRowVersion()       // maps to ROWVERSION SQL type, auto-incremented by DB
    .IsConcurrencyToken(); // include in WHERE clause on UPDATE/DELETE
```

Both lines are required:
- `.IsRowVersion()` alone → maps the column type but does not add it to the WHERE clause
- `.IsConcurrencyToken()` alone → adds to WHERE clause but does not auto-increment
- Together → full optimistic concurrency behavior

**What EF generates at runtime:**

```sql
-- Without concurrency token:
UPDATE WorkOrders SET Status = @status WHERE Id = @id

-- With concurrency token:
UPDATE WorkOrders SET Status = @status WHERE Id = @id AND RowVersion = @originalRowVersion
-- If 0 rows affected → DbUpdateConcurrencyException
```

---

### Layer 3 — API: ExceptionHandlingMiddleware

File: `src/CleanArchitecture.Cmms.Api/Middlewares/ExceptionHandlingMiddleware.cs`

```csharp
case DbUpdateConcurrencyException concurrencyEx:
    await WriteConcurrencyResponseAsync(context, concurrencyEx);
    break;

private async Task WriteConcurrencyResponseAsync(HttpContext context, DbUpdateConcurrencyException ex)
{
    _logger.LogWarning(ex, "Concurrency conflict occurred.");

    var result = Result.Failure(AssetErrors.ConcurrencyConflict);

    context.Response.StatusCode = (int)HttpStatusCode.Conflict; // 409
    await context.Response.WriteAsync(JsonSerializer.Serialize(result, _jsonOptions));
}
```

The exception propagates from `SaveChangesAsync` → `TransactionCommandPipeline` catches it → rolls back → re-throws → `ExceptionHandlingMiddleware` catches it → returns 409.

---

### Known Design Flaw in This Repo

`WriteConcurrencyResponseAsync` always returns `AssetErrors.ConcurrencyConflict`, even if the conflict was on a `WorkOrder` or `Technician`.

The error message would say "Asset was modified by another user" when the actual conflict was on a work order. This is a real bug — a contribution opportunity.

**The fix:** create a generic `ConcurrencyConflict` error not tied to any domain entity, and use it in the middleware. Or inspect `ex.Entries` to identify which entity conflicted and return the appropriate error.

---

## Full Execution Flow: Conflict Scenario

```
Process A: reads Asset X           (RowVersion: 0x0123)
Process B: reads Asset X           (RowVersion: 0x0123)

Process A:
  workOrder.Complete()
    → WorkOrderCompletedEvent raised
    → Asset handler: asset.CompleteMaintenance()
    → [staged in EF change tracker, not yet in DB]
  SaveChangesAsync()
    → UPDATE Assets SET Status='Active' WHERE Id=X AND RowVersion=0x0123
    → 1 row affected ✓
    → RowVersion automatically becomes 0x0124
  CommitAsync() ✓

Process B (slightly later, same transaction):
  workOrder.Complete()
    → WorkOrderCompletedEvent raised
    → Asset handler: asset.CompleteMaintenance()
    → [staged in EF change tracker]
  SaveChangesAsync()
    → UPDATE Assets SET Status='Active' WHERE Id=X AND RowVersion=0x0123
    → 0 rows affected ✗ (RowVersion is now 0x0124, not 0x0123)
    → DbUpdateConcurrencyException thrown
  TransactionCommandPipeline catches exception
    → RollbackAsync()
    → Re-throws exception
  ExceptionHandlingMiddleware catches exception
    → Returns HTTP 409 Conflict
```

---

## Use Cases: When Optimistic Concurrency Is the Right Tool

### 1. Two users edit the same invoice simultaneously
- User A adds a line item and saves
- User B tries to post the invoice
- B gets 409 → refreshes → sees the updated total → posts correctly
- **Optimistic concurrency is sufficient.** Both operations are on the same `Invoice` aggregate.

### 2. Two work orders created for the same asset
- CMMS scenario from ADR-002
- Both read Asset (Status: Active)
- Both create a work order and trigger `WorkOrderCreatedEvent`
- Both event handlers try to update the same Asset
- The second one gets `DbUpdateConcurrencyException` → rolls back entirely
- **Optimistic concurrency on Asset is sufficient.** The invariant ("asset can only have one active work order") lives on the Asset aggregate.

### 3. Two technicians assigned to the same work order
- Two dispatchers assign different technicians simultaneously
- Both read WorkOrder with same RowVersion
- First save succeeds, second gets 409
- Dispatcher two refreshes and sees the assignment already made
- **Optimistic concurrency is sufficient.**

### 4. Concurrent stock reductions (same product)
- Two sales reduce stock of Product X simultaneously
- Both read StockItem with same RowVersion
- First reduces 30 → succeeds
- Second tries to reduce 80 → 0 rows affected → 409
- Second user refreshes → sees remaining quantity (70) → decides if 80 is still valid
- **Optimistic concurrency is sufficient** IF the invariant (available stock) lives on `StockItem`.

---

## Use Cases: When Optimistic Concurrency Is NOT Enough

Optimistic concurrency protects **one aggregate at a time**. It cannot protect a business rule that spans two different aggregate records that are updated in separate transactions.

### Scenario 1: Check-then-act on a different aggregate

**Problem:** You read data from Aggregate A to validate a write to Aggregate B. Optimistic concurrency on Aggregate B does not protect against Aggregate A changing between your check and your write.

**SA Example:**
```
Rule: An invoice can only be posted if the customer's credit limit allows it.

Time 1: User A reads CustomerAccount (Balance = 8,000, Limit = 10,000)
Time 2: User B reads CustomerAccount (Balance = 8,000, Limit = 10,000)
Time 3: User A checks Invoice A (1,500) → 8,000 + 1,500 = 9,500 < 10,000 → PASS
Time 4: User B checks Invoice B (1,800) → 8,000 + 1,800 = 9,800 < 10,000 → PASS
Time 5: User A posts Invoice A → updates CustomerAccount Balance to 9,500
Time 6: User B posts Invoice B → updates CustomerAccount Balance...
```

If both invoice postings update `CustomerAccount` via event handlers → the second event handler tries to update `CustomerAccount` with a stale RowVersion → **409 thrown, Invoice B rolls back.**

**Optimistic concurrency IS enough here** because both write to the same `CustomerAccount` aggregate. The second event handler detects the conflict.

**But this breaks down when:** the credit check happens inside `Invoice.Post()` using data passed in, and the CustomerAccount is never written to in the same transaction. Then two invoices can both pass the check and both commit.

**Workaround:** The event handler that updates `CustomerAccount` must always load and save `CustomerAccount`, even if just to increment the balance. This ensures RowVersion conflict detection catches it.

---

### Scenario 2: Seasonal budget constraint across multiple farmer payments

**Problem:** You have a total seasonal payout budget (e.g., 10,000 JOD). Multiple farmers are paid simultaneously. Each payment is to a different `FarmerPayment` aggregate — different rows, different RowVersions. No single aggregate owns the budget constraint.

```
SeasonBudget = 10,000
Farmer A payment: 6,000 → reads budget 10,000 remaining → PASS
Farmer B payment: 5,000 → reads budget 10,000 remaining → PASS
Both commit → total paid = 11,000 → budget violated
```

Optimistic concurrency on `FarmerPayment` does nothing here because A and B are different rows. Neither conflicts with the other.

**Workaround options:**

**Option A — Create a SeasonBudget aggregate:**
```
SeasonBudget owns the constraint.
Every farmer payment reads AND writes to SeasonBudget.
SeasonBudget.AllocateFunds(amount) validates the rule.
The second payment's event handler hits a RowVersion conflict on SeasonBudget → rolls back.
```

**Option B — Database constraint:**
```sql
-- A CHECK constraint or a trigger that rejects updates exceeding the total
ALTER TABLE SeasonBudgets
ADD CONSTRAINT CK_Budget_NotExceeded CHECK (TotalPaid <= BudgetLimit)
```
The DB enforces the rule at the storage level, regardless of application concurrency.

**Option C — Pessimistic lock on SeasonBudget for the critical section:**
```csharp
// Lock the budget row for the duration of the payment transaction
var budget = await _context.SeasonBudgets
    .FromSqlRaw("SELECT * FROM SeasonBudgets WITH (UPDLOCK) WHERE Id = @id", id)
    .FirstAsync();
```
Used only for this specific critical allocation — not system-wide pessimistic locking.

---

### Scenario 3: Unique constraint that optimistic concurrency cannot express

**Problem:** Two users create a new invoice for the same customer on the same date with the same reference number. These are two INSERT operations — no existing row to conflict on. RowVersion only exists after the row is created.

**Workaround:** Database UNIQUE INDEX.
```sql
CREATE UNIQUE INDEX UX_Invoice_CustomerRef
ON Invoices (CustomerId, ReferenceNumber, InvoiceDate)
```
The second insert fails with a SQL unique constraint violation. Catch it at the application layer and return a meaningful error.

---

### Scenario 4: Long-running business processes (multi-step workflows)

**Problem:** An agricultural auction runs for 2 hours. Bidders read the current lot, submit bids, and the state evolves. RowVersion conflicts would cause 409 errors constantly under high concurrency — not a race condition, but expected concurrent writes.

**Workaround:** Domain-specific event sourcing or reservation pattern.
- Instead of updating the lot directly, each bid is an INSERT (append-only event)
- The current state is derived from the sequence of bids
- No RowVersion conflicts because you never update a shared row

---

## Application-Level Versioning — Why It Was Rejected

An alternative to `ROWVERSION` is maintaining your own version counter as an integer column, incremented by the application.

```csharp
public int Version { get; private set; }

public void IncrementVersion() => Version++;
```

**Problems:**
- You must remember to increment it every time you modify the aggregate
- Under high concurrency, two transactions can both read `Version = 5`, both increment to 6, and EF still sees a conflict — but your increment is now wrong
- More code, more places for bugs
- No benefit over the database-managed `ROWVERSION`

**The database is better at this.** `ROWVERSION` is atomic, automatic, and managed at the storage engine level. Never fight the database for work it can do better.

---

## What Happens to the Transaction When a Conflict Occurs

```
ProcessDomainEventsInBatches completes (all handlers ran in memory)
  ↓
TransactionCommandPipeline: SaveChangesAsync()
  ↓
EF Core sends UPDATE with RowVersion WHERE clause
  ↓
SQL Server: 0 rows affected
  ↓
EF Core throws DbUpdateConcurrencyException
  ↓
SaveChangesAsync propagates exception
  ↓
TransactionCommandPipeline catch block:
  await transaction.RollbackAsync()  ← all staged changes undone
  throw                              ← re-throws exception up the stack
  ↓
ExceptionHandlingMiddleware:
  catches DbUpdateConcurrencyException
  returns HTTP 409 Conflict
  ↓
Client receives: { "error": "Asset was modified by another user. Please refresh." }
```

**Nothing was written to the database.** SaveChangesAsync never completed. The rollback is a safety net but technically the DB was never changed.

---

## Retry Strategy: When to Retry vs When to Force User Intervention

### Safe to auto-retry
Operations where the outcome does not depend on the stale data the user was looking at.

Example: background jobs, idempotent operations, status flag updates where the only possible states are known.

### Must force user to intervene
Any operation in a financial system where the user made a decision based on data they read — and that data has now changed.

**SA rule of thumb:**
- If the user typed a number (quantity, amount, price) → **force refresh.** Their number is now invalid.
- If the system is just flipping a status flag and the data hasn't changed in a meaningful way → consider retry.
- When in doubt → **force refresh.** Wrong data in a financial system is worse than an inconvenient UX.

---

## SA System Summary: Where RowVersion Protects You

| Scenario | Protected? | Why |
|----------|-----------|-----|
| Two users edit same invoice | Yes | Same Invoice aggregate → RowVersion conflict on second save |
| Two invoices posted for same customer | Yes (if event handler writes CustomerAccount) | CustomerAccount aggregate updated by both handlers → second gets conflict |
| Two work orders for same asset | Yes | Asset aggregate updated by both WorkOrderCreated handlers |
| Two farmer payments simultaneously | Only if SeasonBudget aggregate exists | Different FarmerPayment rows — no shared aggregate to conflict on |
| Two new invoices with same reference | No | INSERT operations — no existing RowVersion to conflict |
| Stock oversell (different products) | Yes per product | Each StockItem is its own aggregate |
| Stock oversell (same product, race) | Yes | Same StockItem aggregate → second update conflicts |

---

## Three Rules to Remember

```
1. RowVersion = auto-managed by SQL Server, changes on every UPDATE.
   EF includes it in WHERE clause → 0 rows affected → exception.

2. Conflict detected at SaveChangesAsync — never at read, never at business logic.
   HTTP 409 Conflict — not 400, not 500.

3. Optimistic concurrency protects one aggregate at a time.
   If the invariant lives on a different aggregate than the one you're updating,
   you must protect the aggregate that OWNS the invariant — not the one you're modifying.
```

---

## Quick Reference: Implementation Checklist

```csharp
// 1. Base class (already done in this repo)
[Timestamp]
public byte[] RowVersion { get; protected set; } = default!;

// 2. EF configuration (already done)
builder.Property(e => e.RowVersion)
    .IsRowVersion()
    .IsConcurrencyToken();

// 3. Exception handling (already done — but has the AssetErrors bug)
case DbUpdateConcurrencyException:
    return HTTP 409 Conflict;

// 4. Client behavior
// On 409: show "record was modified, please refresh" — never auto-retry financial ops

// 5. Cross-aggregate invariants
// Identify which aggregate OWNS the rule.
// Ensure that aggregate is always read AND written in the transaction.
// That triggers the RowVersion check on the right record.
```
