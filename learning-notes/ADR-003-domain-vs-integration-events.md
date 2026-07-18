# ADR-003: Domain Events vs Integration Events

> Personal learning notes — Noor Tantawi  
> Source repo: CleanArchitecture-DDD-CQRS (open source CMMS)

---

## The Anchor

**Domain Event Handler** = An internal emergency meeting between department heads, held immediately when something happens. Finance, Inventory, and Operations are all in the same room at the same time. If one person cannot complete their part, the meeting is cancelled and nothing is recorded. Everything happens together or not at all.

**Integration Event Handler** = A letter you send to someone outside your company *after* the internal meeting is done and everything is filed. If the post office loses it, you resend it. You do not un-file your internal records because of a postal failure.

**The core insight:** The same domain event can have BOTH types of handlers. One event — two categories of reaction.

---

## The Problem

In ADR-001 we established that domain events coordinate cross-aggregate changes. But not all reactions to an event need the same guarantees.

`WorkOrderCompletedEvent` example:

| Reaction | Must be in same transaction? | If it fails? |
|----------|------------------------------|-------------|
| Asset → return to Active | **Yes.** Completed work order with asset still "Under Maintenance" is a broken business state. | Roll back the work order completion. |
| Send notification email | **No.** A delayed email is acceptable. | Retry. Work order stays completed. |

Treating both the same forces a bad choice:
- **All synchronous** → email provider goes down at 2 AM → every work order completion rolls back
- **All async** → asset status becomes eventually consistent → business rules temporarily violated

---

## The 4 Options

### Option 1 — Everything Synchronous (Rejected)
All events run inside the transaction. Email, external APIs, audit logs — all transactional.

**Problem:** External service failure destroys core business operation. Email timeout = transaction rollback = work order not completed. Unacceptable.

### Option 2 — Everything Asynchronous (Rejected)
All events run via outbox after commit. Even critical state changes happen eventually.

**Problem:** Window of inconsistency between aggregates. Race conditions. Business invariants temporarily violated. Unacceptable for financial/critical operations.

### Option 3 — Two Distinct Interfaces (Chosen)
`IDomainEventHandler` for strong consistency inside the transaction.
`IIntegrationEventHandler` for eventual consistency outside the transaction.

### Option 4 — Single Interface with Attributes (Rejected)
```csharp
[Transactional]
public class AssetHandler : IEventHandler<WorkOrderCompleted> { }
```
**Problem:** Developer forgets the attribute. No compile-time safety. Runtime error in production.

---

## The Decision: Two Interfaces, Two Guarantees

### IDomainEventHandler\<TEvent\>

```csharp
public interface IDomainEventHandler<in TEvent> where TEvent : IDomainEvent
{
    Task Handle(TEvent domainEvent, CancellationToken cancellationToken = default);
}
```

**Contract:**
- Runs **inside** the database transaction
- Can load and modify aggregates via repositories
- Changes commit atomically with the command
- If it throws → **full transaction rollback**
- No retry — succeeds with everything or fails with everything

**Use when:**
- Handler modifies aggregate state
- Requires immediate consistency
- Failure must prevent the core operation from completing

---

### IIntegrationEventHandler\<TEvent\>

```csharp
public interface IIntegrationEventHandler<in TEvent>
{
    Task Handle(TEvent @event, CancellationToken cancellationToken = default);
}
```

**Contract:**
- Runs **outside** the transaction, after it commits
- Invoked by background outbox processor
- Guaranteed delivery — outbox entry was written inside the transaction
- If it throws → **retry** (outbox entry stays, retried later)
- Core business operation unaffected by failure

**Use when:**
- Sends notifications (email, SMS, WhatsApp, push)
- Calls external APIs or services
- Writes audit logs or analytics
- Updates read models or caches
- Publishes to message bus
- Can tolerate eventual consistency

---

## Critical Distinction: Repository vs Handler Interface

These are different things. Do not confuse them:

```csharp
// IFarmerAccountRepository = what you INJECT into the handler
// IDomainEventHandler = what the handler IS

public class FarmerPaymentObligationHandler
    : IDomainEventHandler<SaleOrderConfirmedEvent>   // ← handler type
{
    private readonly IFarmerAccountRepository _repo; // ← injected dependency

    public async Task Handle(SaleOrderConfirmedEvent @event, CancellationToken ct)
    {
        var obligation = await _repo.GetByFarmerId(@event.FarmerId, ct);
        obligation.RecordOwed(@event.Amount);
    }
}
```

---

## Where Each Handler Lives in This Repo

```
src/CleanArchitecture.Cmms.Application/

  Assets/Events/WorkOrderCompleted/
    WorkOrderCompletedEventHandler.cs    ← IDomainEventHandler  (modifies Asset)

  Technicians/Events/WorkOrderCompleted/
    WorkOrderCompletedEventHandler.cs    ← IDomainEventHandler  (modifies Technician)

  Integrations/Events/WorkOrderCompleted/
    EmailWorkOrderCompletedHandler.cs    ← IIntegrationEventHandler  (sends email)
```

Same event (`WorkOrderCompletedEvent`), three handlers, two different interfaces.

---

## Full Execution Flow

```
workOrder.Complete()
  → raises WorkOrderCompletedEvent (in memory)

DomainEventsPipeline dispatches to IDomainEventHandlers [inside transaction]:
  ├─ Assets.WorkOrderCompletedEventHandler
  │    asset.CompleteMaintenance(...)
  └─ Technicians.WorkOrderCompletedEventHandler
       technician.CompleteAssignment(...)

DomainEventsPipeline writes to Outbox [inside same transaction]:
  └─ OutboxMessage { EventType = "WorkOrderCompletedEvent", Payload = ... }

SaveChangesAsync()   ← all domain handler changes + outbox entry committed atomically
CommitAsync()

--- transaction boundary ---

Background Outbox Processor (separate process, runs later):
  reads OutboxMessage
  dispatches to IIntegrationEventHandlers:
  └─ EmailWorkOrderCompletedHandler
       sends email
       if fails → outbox entry stays → retry next cycle
       work order stays completed regardless
```

---

## Decision Flowchart

```
New handler needed. Which interface?

Does it modify aggregate state?
├── YES → IDomainEventHandler
│         (inside transaction, failure = rollback)
│
└── NO  → Does failure affect business correctness?
           ├── YES → IDomainEventHandler
           │         (e.g. credit check — failure must prevent operation)
           │
           └── NO  → IIntegrationEventHandler
                      (email, SMS, audit, cache, external API, analytics)
```

---

## SA System Examples

| Event | Handler | Type | Reason |
|-------|---------|------|--------|
| `InvoicePostedEvent` | Update `CustomerAccount` balance | `IDomainEventHandler` | Aggregate state, must be atomic |
| `InvoicePostedEvent` | Create `JournalEntry` | `IDomainEventHandler` | Accounting consistency required |
| `InvoicePostedEvent` | Reduce `InventoryItem` stock | `IDomainEventHandler` | Inventory consistency required |
| `InvoicePostedEvent` | Email invoice to buyer | `IIntegrationEventHandler` | External call, failure acceptable, retry via outbox |
| `InvoicePostedEvent` | Write audit log | `IIntegrationEventHandler` | Not business-critical, eventual consistency fine |
| `PaymentReceivedEvent` | Mark `Invoice` as paid | `IDomainEventHandler` | Aggregate state change, must be atomic |
| `PaymentReceivedEvent` | Send SMS to farmer | `IIntegrationEventHandler` | Notification, external system, retry acceptable |
| `SaleOrderConfirmedEvent` | Record `FarmerPaymentObligation` | `IDomainEventHandler` | Aggregate state change, must be transactional |
| `SaleOrderConfirmedEvent` | WhatsApp notification to buyer | `IIntegrationEventHandler` | External notification, eventual consistency fine |
| `SaleOrderConfirmedEvent` | Report to government tax API | `IIntegrationEventHandler` | External API, retry with idempotency key |
| `InvoicePostedEvent` | Update analytics dashboard | `IIntegrationEventHandler` | Non-critical, failure must not affect balance posting |

---

## The Government Tax API — Special Case

Reporting a financial transaction to an external government API requires careful design.

**Wrong approach:** Call the government API inside the transaction (IDomainEventHandler).
- Transaction stays open during network I/O
- API timeout = transaction rollback = invoice not posted (critical business operation killed)
- Government may have already received and processed the request before timeout

**Correct approach:** `IIntegrationEventHandler` with an idempotency key.

```csharp
public class GovernmentTaxHandler
    : IIntegrationEventHandler<InvoicePostedEvent>
{
    public async Task Handle(InvoicePostedEvent @event, CancellationToken ct)
    {
        await _taxApi.ReportTransaction(new TaxReportRequest
        {
            IdempotencyKey = @event.InvoiceId.ToString(), // ← prevents double-reporting on retry
            Amount = @event.Total,
            Date = @event.PostedAt
        });
    }
}
```

**Idempotency key:** Include your internal transaction ID in every API call. If the outbox retries the call three times due to timeouts, the government API recognizes the duplicate and ignores it. Exactly-once semantics even with at-least-once delivery.

**What about government rollback?** External systems do not participate in your database transaction. If an invoice must be cancelled after it was reported, you issue a **compensating transaction** — a credit note, a reversal, a cancellation API call. You never un-do; you correct-forward. This is how all real financial systems work.

---

## One Handler Doing Two Things — Wrong Design

A handler that modifies aggregate state AND calls an external service violates single responsibility and creates fragility.

```csharp
// WRONG: two responsibilities, two different consistency requirements
public class CustomerAccountHandler : IDomainEventHandler<InvoicePostedEvent>
{
    public async Task Handle(InvoicePostedEvent @event, CancellationToken ct)
    {
        var account = await _accountRepo.GetByIdAsync(@event.CustomerId, ct);
        account.IncreaseBalance(@event.Total);   // ← must be in transaction

        await _analyticsApi.Track(@event);       // ← external call, must NOT be in transaction
        // If analytics API fails → balance update rolls back → WRONG
    }
}
```

```csharp
// CORRECT: two handlers, each with one responsibility and one guarantee

public class CustomerAccountHandler : IDomainEventHandler<InvoicePostedEvent>
{
    public async Task Handle(InvoicePostedEvent @event, CancellationToken ct)
    {
        var account = await _accountRepo.GetByIdAsync(@event.CustomerId, ct);
        account.IncreaseBalance(@event.Total);
        // committed atomically with the invoice posting
    }
}

public class AnalyticsDashboardHandler : IIntegrationEventHandler<InvoicePostedEvent>
{
    public async Task Handle(InvoicePostedEvent @event, CancellationToken ct)
    {
        await _analyticsApi.Track(@event);
        // runs after commit, retries if it fails, does not affect balance
    }
}
```

---

## Future Evolution: Zero Business Code Changes

**Today (single process):**
```
Integration event → outbox table → background worker → invokes handler
```

**Tomorrow (microservices):**
```
Integration event → outbox table → outbox processor → RabbitMQ / Azure Service Bus → other service → invokes handler
```

The `IIntegrationEventHandler` implementation does not change. Only infrastructure wiring changes. Domain event handlers remain internal to each service — they never cross service boundaries.

---

## The Compile-Time Safety Point

This is why attribute-based distinction was rejected:

```csharp
// Type system makes the guarantee visible and enforced at compile time:

public class AssetHandler : IDomainEventHandler<WorkOrderCompletedEvent>
// → compiler enforces: this runs in a transaction

public class EmailHandler : IIntegrationEventHandler<WorkOrderCompletedEvent>
// → compiler enforces: this runs via outbox, after commit

// You cannot accidentally put a state-changing handler in the outbox.
// You cannot accidentally put an email handler in the transaction.
```

---

## Three Rules to Remember

```
1. Same event — two handler types. IDomainEventHandler inside the transaction.
   IIntegrationEventHandler outside the transaction via outbox.

2. The handler type IS the consistency guarantee.
   IDomainEventHandler = strong consistency, rollback on failure.
   IIntegrationEventHandler = eventual consistency, retry on failure.

3. Never mix responsibilities in one handler.
   Aggregate state change = IDomainEventHandler.
   External call = IIntegrationEventHandler.
   Two concerns = two separate handlers.
```
