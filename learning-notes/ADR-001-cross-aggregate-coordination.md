# ADR-001: Cross-Aggregate Coordination

> Personal learning notes — Noor Tantawi  
> Source repo: CleanArchitecture-DDD-CQRS (open source CMMS)

---

## The Problem

When one business operation touches **multiple aggregates** across different bounded contexts, how do you coordinate the changes without:
- Coupling your bounded contexts together
- Losing ACID guarantees
- Breaking aggregate boundary rules

**Real-life anchor:** One department head should not walk into another department and edit their records. They send an announcement email. Each department reacts on their own.

---

## Core Vocabulary

### Aggregate
A cluster of objects treated as one consistency unit. Protects its own business rules. Outsiders interact only through the root.

> **Anchor:** A department head who owns their area and enforces all rules within it.

**SA Example:** `Invoice` is an aggregate. It protects rules like "you cannot post an invoice with no lines" and "you cannot change lines after posting."

---

### Aggregate Root
The front door of the aggregate. The only entry point for outsiders.

> **Anchor:** The Finance Manager. You don't touch Finance spreadsheets directly — you go through the manager.

**SA Example:** `Invoice` is the aggregate root. `InvoiceLine` lives inside it. You never update `InvoiceLine` directly — you call `invoice.AddLine(...)` or `invoice.RemoveLine(...)`.

---

### Bounded Context
A business area with its own language and model. The same word can mean different things across contexts.

> **Anchor:** HR, Finance, and Sales each speak their own language. "Account" means a user login in Auth, a customer profile in Sales, and a chart-of-accounts entry in Accounting.

**SA Example:**
- **Sales context:** Invoice, Customer, SalesOrder
- **Accounting context:** JournalEntry, LedgerAccount, FiscalPeriod
- **Inventory context:** StockItem, Reservation, Warehouse

---

### Domain Event
Something meaningful that happened in the business. Named in **past tense** because it already occurred.

> **Anchor:** A company-wide email. "Invoice was posted." Each department decides how to react. Nobody told them what to do — they subscribed.

**SA Examples:** `InvoicePostedEvent`, `PaymentReceivedEvent`, `SalesOrderConfirmedEvent`

Rule: name events in past tense. `InvoicePostedEvent` is correct. `InvoiceUpdatedEvent` is too vague.

---

### Cross-Aggregate Coordination
One use case requires multiple aggregates to change.

> **Anchor:** Posting an invoice affects Invoice, CustomerAccount, JournalEntry, and Inventory — four department heads must act.

---

### Transactional Domain Event Handler
Reacts inside the **same database transaction**. If it fails, everything rolls back.

**SA Example:** When `InvoicePostedEvent` fires, the `AccountingEventHandler` creates a `JournalEntry`. If that fails, the invoice posting rolls back too.

---

### Integration Event Handler
Reacts **after** the transaction commits. Failure does NOT roll back the core operation.

> **Anchor:** The courier picks up your letter after you've already filed the original. If the courier fails, the filing is not undone.

**SA Example:** Sending a posted invoice by email. If the email fails, the invoice stays posted. The email retries via the outbox.

---

### Application Command Handler
Coordinates a use case. Should touch only **its own aggregate**. Does not orchestrate other bounded contexts directly.

> **Anchor:** The receptionist hands your request to one department. Does not run between departments themselves.

**SA Example:** `PostInvoiceCommandHandler` injects only `IRepository<Invoice, Guid>`. It does not inject `IRepository<JournalEntry>` or `IRepository<CustomerAccount>`.

---

## The 5 Options Considered

| Option | What It Does | Verdict |
|--------|-------------|---------|
| **1. Handler Orchestration** | Handler injects all repos and updates all aggregates directly | Acceptable for simple single-context cases. Gets messy fast. |
| **2. Mediator.Send(Command)** | Handler sends commands to other bounded contexts via mediator | Not recommended. Hidden chains, unclear transaction boundaries. |
| **3. Mediator.Send(Query)** | Handler reads data from another context for validation | Acceptable but risky. Race condition between read and write. |
| **4. Domain Service** | Dedicated service coordinates multiple aggregates explicitly | Legitimate DDD. Use for complex coordination or bidirectional data flow. |
| **5. Domain Events** | Aggregate raises event, other contexts react as handlers | **Chosen default.** Clean boundaries, single transaction, testable. |

---

## The Decision

**Default pattern: Domain Events (transactional)**

**Rule:** A command handler injects only repositories from its own bounded context. Other aggregates react to domain events.

**Exception — Domain Service:** When coordination logic is complex, belongs in the domain, and no single aggregate owns the rule.

> Example: Before confirming a large sale, checking credit limit (BuyerAccount) + stock availability (Inventory) + active contract (FarmerContract) simultaneously. No single aggregate owns that rule. That is a domain service.

---

## Execution Flow

```
Command arrives
  ↓
TransactionCommandPipeline → BeginTransaction
  ↓
DomainEventsPipeline → calls CommandHandler
  ↓
CommandHandler → loads ONE aggregate → calls domain method
  → domain method raises event (in memory, not yet in DB)
  ↓
DomainEventsPipeline → ProcessDomainEventsInBatches
  → Batch 1: dispatch all current events to their handlers
      each handler loads its own aggregate
      each handler calls its own domain method
      if handlers raise new events → collected for Batch 2
  → Batch 2, 3... until no more events are raised
  ↓
WriteIntegrationEventsToOutbox (email, notifications — post-commit)
  ↓
SaveChangesAsync → ONE write to database for everything
CommitAsync
  ↓
If anything throws before commit → RollbackAsync
```

**Critical rule:** `SaveChangesAsync` runs once, at the very end. Until that moment, everything is staged in the EF change tracker (in memory). Nothing hits the database until that single moment.

**The transaction rolls back — not "the event."** The event is in-memory. It has no database presence to undo. What rolls back is the database write that never happened.

**Batch processing is breadth-first, not depth-first.** Batch 1 fully completes before Batch 2 starts. If two event handlers each raise a new event, both new events go into Batch 2 together — neither interrupts the other.

---

## Architecture Test That Enforces This

```csharp
[Fact]
public void CommandHandlers_Should_Use_Repository_From_Same_BoundedContext()
{
    // if a command handler injects a repo from another bounded context
    // this test fails at build time
}
```

The rule is enforced in code, not just documentation. You cannot accidentally break it.

---

## SA System Full Example: PostInvoice

### The use case: Post Invoice #1042

**What must happen:**
1. Invoice → status becomes Posted
2. CustomerAccount → balance increases
3. JournalEntry → debit Accounts Receivable, credit Sales Revenue
4. InventoryItem → stock decreases (if applicable)
5. Email → send posted invoice to buyer (after commit, non-transactional)

---

### The command handler (touches only Invoice)

```csharp
internal sealed class PostInvoiceCommandHandler
    : ICommandHandler<PostInvoiceCommand, Result>
{
    private readonly IRepository<Invoice, Guid> _invoiceRepository;

    public PostInvoiceCommandHandler(IRepository<Invoice, Guid> invoiceRepository)
        => _invoiceRepository = invoiceRepository;

    public async Task<Result> Handle(PostInvoiceCommand request, CancellationToken ct)
    {
        var invoice = await _invoiceRepository.GetByIdAsync(request.InvoiceId, ct);
        if (invoice is null) return InvoiceErrors.NotFound;

        invoice.Post(); // protects rules + raises InvoicePostedEvent

        return Result.Success();
    }
    // No IRepository<CustomerAccount>.
    // No IRepository<JournalEntry>.
    // No IRepository<InventoryItem>.
    // Those are handled by their own event handlers.
}
```

---

### Event handlers that react (each in their own bounded context)

```
InvoicePostedEvent dispatched to:

  [Batch 1 — Transactional]
  → Accounting.InvoicePostedEventHandler
      loads JournalEntry aggregate
      journalEntry.Post(debit AR, credit Revenue)

  → CustomerAccount.InvoicePostedEventHandler
      loads CustomerAccount aggregate
      account.IncreaseBalance(invoice.Total)

  → Inventory.InvoicePostedEventHandler
      loads InventoryItem aggregate
      item.DecreaseStock(invoice.Lines)

  [Integration — fires after commit, non-transactional]
  → Email.InvoicePostedHandler
      sends posted invoice to buyer
      failure = retry via outbox, invoice stays posted
```

---

### What rolls back on failure

| Failure | Result |
|---------|--------|
| `invoice.Post()` throws (e.g. no lines) | Full rollback. Nothing changes. |
| JournalEntry handler throws | Full rollback. Invoice not posted, balance unchanged, stock unchanged. |
| Inventory handler throws | Full rollback. Same as above. |
| Email handler fails | Nothing rolls back. Invoice is posted. Email retries via outbox. |

---

## Summary: Three Rules to Remember

1. **Command handler = one aggregate, one bounded context.** Never inject repos from other contexts.
2. **Domain events coordinate the rest,** all inside one database transaction.
3. **Post-commit work (email, audit, notifications) goes through the outbox,** never through the transaction.
