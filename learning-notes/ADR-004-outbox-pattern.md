# ADR-004: Outbox Pattern for Guaranteed Event Delivery

> Personal learning notes — Noor Tantawi
> Source repo: CleanArchitecture-DDD-CQRS (open source CMMS)
> Status: content taught in full. Drill 4 (3 scenarios) still open — see [tracker.md](tracker.md).

---

## The Anchor

Think of the outbox like a **physical outbox tray on an accountant's desk**.

The accountant posts an invoice (database commit). Before leaving the desk, they place the "email this invoice to the buyer" letter in the outbox tray. Even if they get sick and leave immediately, the letter is still in the tray. A courier comes by every 5 minutes, picks up whatever is there, and delivers it. If delivery fails, the letter stays in the tray and the courier tries again next round.

The letter was never lost. It was written at the same time as the invoice. Its fate is tied to the invoice commit, not to whether the courier happened to be available.

---

## The Core Problem: Dual-Write

Any system using both a **database** and an **external delivery mechanism** (email, message bus, government API, SMS gateway) must update two separate systems. These updates cannot be atomic by default.

### Approach 1 — Publish First, Then Commit (Rejected)
```
1. Send email to buyer          ← succeeds
2. Commit invoice to database   ← crashes
Result: email sent for an invoice that doesn't exist in the database
```

### Approach 2 — Commit First, Then Publish (Rejected — the naive default)
```
1. Commit invoice to database   ← succeeds
2. Send email to buyer          ← app crashes before this line
Result: invoice exists, email never sent, no record it should have been
```
This is what most systems do by accident. Every server restart, deployment, or exception risks silently losing these notifications forever.

### Approach 3 — Two-Phase Commit (Rejected)
Lock both the database and the message broker, commit both atomically.
Problem: complex, poor performance, requires a distributed transaction coordinator. No modern web system uses this for this purpose.

**The outbox solves all three** by turning the database itself into a temporary message queue.

---

## The Solution: Transactional Outbox Pattern

```
Business transaction:
  1. Post invoice (update Invoices table)
  2. Write outbox message (insert into OutboxMessages table)
  3. Commit ← both writes atomic, in ONE transaction

Background processor (separate, runs every 5 seconds):
  4. Read next unprocessed outbox message
  5. Deserialize and dispatch to IIntegrationEventHandler
  6. Mark message as ProcessedAt = now
  7. If handler fails → increment RetryCount, try again next cycle
```

Step 3 is the key. The outbox entry is written in the **same transaction** as the invoice. If the transaction commits, the outbox entry is guaranteed to exist. If it rolls back, the outbox entry is also gone. They are always in sync.

---

## The 4 Options Considered

| Option | Pattern | Verdict |
|--------|---------|---------|
| 1. Fire-and-forget (`Task.Run`) | Background task, no persistence | Rejected — events lost on crash, no retry, no delivery guarantee |
| 2. In-memory queue + background service | Queue in memory, worker processes it | Rejected — queue lost on restart, not restart-safe |
| **3. Transactional Outbox (Chosen)** | Event row written in same DB transaction as business data | Accepted — guaranteed delivery, survives restarts, at-least-once |
| 4. Two-Phase Commit | Distributed transaction coordinator | Rejected in Approach 3 above — too complex, poor performance |

---

## The OutboxMessage Schema

```csharp
public sealed class OutboxMessage
{
    public Guid Id { get; set; }
    public string EventType { get; set; }      // full type name, used for deserialization
    public string Payload { get; set; }        // JSON of the event
    public DateTime CreatedAt { get; set; }
    public DateTime? ProcessedAt { get; set; } // null = not yet processed
    public int RetryCount { get; set; }
    public string? LastError { get; set; }     // last failure reason
    public int MaxRetries { get; set; } = 3;
}
```

**SA equivalent — a row after posting Invoice #1042:**
```
Id:          a3f1c2d4-...
EventType:   SalesAndAccounting.Domain.Events.InvoicePostedEvent, SA.Domain
Payload:     {"invoiceId":"1042","customerId":"C-001","total":2500.00,"postedAt":"2026-06-28T..."}
CreatedAt:   2026-06-28 10:30:00
ProcessedAt: null               ← not yet delivered
RetryCount:  0
LastError:   null
MaxRetries:  3
```

---

## The Background Processor

`OutboxProcessor` is a `BackgroundService` that starts with the app and loops forever, polling every 5 seconds. `OutboxMessageProcessor` does the actual work, up to 10 messages per cycle:

```
For each unprocessed message:
  1. Lock the row (UPDLOCK, ROWLOCK) — prevents two workers processing the same message
  2. Skip locked rows (READPAST) — lets multiple workers run in parallel safely
  3. Deserialize Payload back into the original event type
  4. Dispatch to IIntegrationEventDispatcher → finds and calls the right handler(s)
  5. Success → MarkAsProcessedAsync (ProcessedAt = now), commit
  6. Failure → IncrementRetryAsync (RetryCount++, LastError = message), commit
  7. RetryCount >= MaxRetries → moved to Dead Letter table
```

**Why `WITH (UPDLOCK, ROWLOCK, READPAST)` matters:**
- `UPDLOCK` — lock this row so no other worker can claim it
- `ROWLOCK` — lock only this row, not the whole table
- `READPAST` — if a row is locked by another worker, skip it instead of waiting

This is what allows multiple `OutboxProcessor` instances to run simultaneously without stepping on each other — critical once you scale to multiple app instances/pods.

---

## SA System: Full Lifecycle Example

**Scenario:** Buyer pays Invoice #1042. Payment = 2,500 JOD.

### Step 1 — Transaction (everything in one commit)
```
PaymentReceivedCommand
  → PaymentReceivedCommandHandler
      payment = Payment.Create(invoiceId, amount)
      payment.Receive() → raises PaymentReceivedEvent

DomainEventsPipeline — IDomainEventHandlers [inside transaction]:
  → Invoice.InvoicePaymentEventHandler
       invoice.MarkAsPaid()   ← aggregate state, must be atomic

DomainEventsPipeline — writes to OutboxMessages [same transaction]:
  → Row 1: { EventType: PaymentReceivedEvent, Payload: {...}, ProcessedAt: null }

SaveChangesAsync → Payment + Invoice status + OutboxMessage all committed together
CommitAsync ✓
```

### Step 2 — Background processor (after commit, separate)
```
5 seconds later, OutboxProcessor wakes up:
  Reads Row 1 (ProcessedAt IS NULL)
  Deserializes Payload → PaymentReceivedEvent
  Dispatches to IIntegrationEventHandlers:
    → SMSNotificationHandler       sends "Payment received" SMS to farmer
    → AnalyticsDashboardHandler    updates revenue dashboard
    → GovernmentTaxHandler         reports transaction to tax API
  All succeed → ProcessedAt = now → committed
```

### Step 3 — What if the SMS gateway is down?
```
Attempt 1: SMSNotificationHandler throws "Gateway unavailable"
           RetryCount = 1, LastError = "Gateway unavailable"
           Other handlers (analytics, tax) still succeed on their own delivery
5s later, Attempt 2: same failure → RetryCount = 2
5s later, Attempt 3: gateway back online → success → ProcessedAt = now

Payment was always posted. Farmer received SMS 15 seconds late.
Business never stopped.
```

### Step 4 — What if the app crashes right after commit?
```
10:30:00  Invoice posted, OutboxMessage written and committed
10:30:01  App crashes (deployment, server restart, power cut)
          OutboxMessage still in database: ProcessedAt = null
10:35:00  App restarts, OutboxProcessor starts
10:35:05  Reads unprocessed message from 5 minutes ago
          Dispatches handlers → email/SMS/tax report all sent
          ProcessedAt = now
```
The event was never lost. It sat in the database the entire time.

---

## At-Least-Once Delivery and Idempotency

The outbox guarantees **at-least-once delivery** — an event will be processed a minimum of one time. It does NOT guarantee exactly-once.

**When can an event be processed twice?**
```
10:30:00  OutboxProcessor dispatches EmailHandler → email sent ✓
10:30:01  App crashes before MarkAsProcessedAsync is called
          ProcessedAt is still null
10:35:05  App restarts, OutboxProcessor reads the SAME message again
          EmailHandler runs again → second email sent to buyer
```

**The fix: idempotency in the handler.**
```csharp
public class InvoiceEmailHandler : IIntegrationEventHandler<InvoicePostedEvent>
{
    public async Task Handle(InvoicePostedEvent @event, CancellationToken ct)
    {
        var alreadySent = await _emailLog.ExistsAsync(@event.InvoiceId, ct);
        if (alreadySent) return;              // ← idempotency check

        await _emailService.SendInvoiceToCustomer(@event.InvoiceId, ct);
        await _emailLog.RecordSentAsync(@event.InvoiceId, ct);
    }
}
```

**Rule:** every `IIntegrationEventHandler` must be written so that running it twice with the same event produces the same result as running it once.

---

## Dead Letter Queue

When `RetryCount >= MaxRetries` (default 3), the message moves to `DeadLetterMessages`:

```csharp
if (message.RetryCount >= message.MaxRetries)
{
    MoveToDeadLetterAsync(message); // moves row to DeadLetterMessages table
}
```

A dead letter means: "we tried 3 times and could not deliver this — a human must investigate."

**SA example:** Government tax API rejects your request because the invoice number format changed. Three retries fail. The tax report goes to dead letter. You:
1. Fix the invoice number format
2. Manually replay the event (re-insert into OutboxMessages, or directly invoke the handler)
3. Clear the dead letter entry

Dead letters are not failures — they are **deferred problems waiting for human judgment**.

**How fast does exhaustion actually happen?** Exhaustion is governed by `RetryCount >= MaxRetries` — a count, not a timer. Inside `OutboxMessageProcessor.ProcessOutboxMessagesAsync`, the `while (processedCount < batchSize)` loop has **no delay between iterations** — the 5-second `Task.Delay` only happens between *calls* to this method (in `OutboxProcessor.ExecuteAsync`), not between individual message attempts inside it. And `GetNextUnprocessedMessageAsync` always orders `BY CreatedAt`, so the same oldest still-eligible message gets picked again immediately on the next iteration. In practice this means a message can burn through all 3 retries back-to-back, within a single invocation, in whatever time 3 failed API calls actually take — which could be milliseconds, or longer if each call hangs until an HTTP timeout. **This is NOT spread out over minutes by design — there is no delay between retry attempts at all**, only between processing cycles.

**The real gap:** there is no exponential backoff. For an outage measured in hours (a government API down for a bank holiday), a fixed "3 attempts with no growing delay" policy means every affected message dead-letters within minutes of the outage starting — long before the outage itself resolves. ADR-004 lists "Exponential backoff" explicitly as a **Future** improvement, confirming this isn't built yet. A production-grade retry policy would space attempts increasingly further apart (1 min, 5 min, 30 min...) so that outages shorter than the backoff ceiling self-heal without ever needing a human to manually resend anything from the dead letter table.

---

## A Real Design Note Worth Knowing — CONFIRMED BUG

`EfCoreOutboxStore.AddAsync` calls `SaveChangesAsync()` internally using a **separate `OutboxDbContext`**, distinct from the main business `DbContext` managed by `TransactionCommandPipeline`.

For the outbox write to truly be atomic with the business data, both contexts must share the same database connection/transaction. **Confirmed by reading the actual registration and transaction code:** `EfUnitOfWork.BeginTransactionAsync` opens a transaction on the main `DbContext` only, and `OutboxDbContext` is registered independently via `AddDbContext<OutboxDbContext>` with its own connection string — no `UseTransaction(...)` call, no shared connection anywhere. This means the outbox write commits **immediately and independently**, before the main transaction even reaches its own commit. If the main transaction later rolls back, the outbox row survives anyway. Full writeup with the exact fix options: [bugs-found.md, Bug #2](bugs-found.md).

---

## Guidelines for Writing Integration Event Handlers

- Must be idempotent (same event processed multiple times → same effect)
- Should not throw for *expected* failures — log and return gracefully where appropriate
- Should log errors for investigation
- Should avoid modifying core aggregate state (that's `IDomainEventHandler`'s job)
- Events should be immutable, self-contained data contracts — include everything the handler needs, avoid requiring extra DB lookups just to reconstruct context

---

## Three Rules to Remember

```
1. The outbox turns "dual-write" into "single-write" by writing the event
   as a row in the SAME transaction as the business data.

2. Delivery guarantee = at-least-once, not exactly-once.
   Every integration handler MUST be idempotent.

3. Retries live in the outbox row itself (RetryCount, LastError).
   Exceeding MaxRetries = Dead Letter = human investigation, not silent failure.
```

---

## Drill 4 — Answered (2026-07-18)

**1. Power loss at 11:00:01, restart next morning.**
Payment record + outbox message row: both durably committed at 11:00 PM, before the crash — unaffected by the power loss. SMS and tax report: neither sent that night (`OutboxProcessor` is in-process and died with the app). On restart, `OutboxProcessor` immediately picks up the still-unprocessed row and dispatches to every registered `IIntegrationEventHandler` for that event — both SMS and tax report go out, ~8 hours late. **Nothing was lost — only delayed.** That's the entire guarantee the outbox provides.

**2. No idempotency check, handler crashes before `MarkAsProcessedAsync`, retried next day.**
Buyer receives two identical emails. Fix: idempotency check in the handler (see `InvoiceEmailHandler` example above) — check an email log before sending, record after sending.

**3. Government tax API down 6 hours, 80 invoices processed during the outage.**
Exhaustion is governed by `RetryCount >= MaxRetries` (a count), not a timer — and there's no delay between individual retry attempts within the processing loop, only between processing cycles. All 80 messages exhaust their 3 retries within minutes of the outage starting, long before the 6-hour outage ends. By morning: **check `DeadLetterMessages`, not `OutboxMessages`** — the messages have already been moved out. Filter by `EventType` + `MovedToDeadLetterAt` falling in the outage window; check `LastError` (preserved) for the actual failure reason and `RetryCount = 3` to confirm legitimate exhaustion. **The real gap this exposes:** no exponential backoff exists yet (ADR-004 lists it as a "Future" item) — a fixed 3-attempts-no-backoff policy is inadequate for hour-scale outages; a production system would space retries increasingly further apart so shorter outages self-heal without any dead-letter/manual-resend step at all.
