# Questions Log

> Every question I ask gets reframed and answered here in full, going forward.
> This is the "extra 10 miles" file — full depth, not summaries.

---

## Q: What is "attribute-based distinction" (ADR-003, Option 4), and why was it rejected in favor of two separate interfaces?

**Reframed:** ADR-003 mentions a rejected design where a single `IEventHandler<T>` interface would be tagged with `[Transactional]` or `[Asynchronous]` attributes to indicate how it should run. What does that actually mean, and why does using two distinct interfaces (`IDomainEventHandler` / `IIntegrationEventHandler`) provide "compile-time safety" that the attribute approach doesn't?

**Answer:**
A C# attribute (`[Transactional]`) is just metadata attached to a class — it does nothing by itself. Something else has to use reflection at runtime to read it and decide what to do. In the rejected design, one interface (`IEventHandler<T>`) would serve all handlers, and a dispatcher would inspect each handler's attributes to decide whether to run it inside the transaction or via the outbox.

The failure mode: nothing stops you from writing a handler and forgetting to add the attribute. It compiles fine — no error, no warning — and the mistake is only discovered at runtime, potentially in production, when an email handler ends up holding a database lock, or a critical state-change handler gets silently treated as fire-and-forget.

Two separate interfaces fix this because "which category a handler belongs to" is encoded directly in the class declaration itself (`: IDomainEventHandler<T>` vs `: IIntegrationEventHandler<T>`) — the part the compiler actually checks. You cannot compile a class claiming to implement an interface without fulfilling its contract, and the interface it declares is permanent, structural, and impossible to "forget." Attributes are optional metadata checked at runtime (if someone remembers to check); interfaces are contracts enforced by the compiler, every time.

*(See [ADR-003 notes](ADR-003-domain-vs-integration-events.md) for full context.)*

---

## Q: What is the main difference between `IComparable` and `IDisposable`?

**Reframed:** These were used as an example of "two interfaces with no shared meaning." What does each actually do, and why are they a good example of unrelated contracts?

**Answer:**
```csharp
public interface IComparable<in T> { int CompareTo(T? other); }
public interface IDisposable { void Dispose(); }
```
`IComparable<T>` defines relative ordering between two instances of a type — `CompareTo` returns negative/zero/positive, powering `Sort()`, `OrderBy`, `SortedSet`. `IDisposable` defines resource cleanup — `Dispose()` releases unmanaged resources (file handles, DB connections, sockets), powering the `using` statement.

They solve completely unrelated problems: "how do two objects rank against each other" vs "how does one object let go of resources it's holding." There's no conceptual bridge between them — which is exactly why they're a good analogy for `IDomainEventHandler` vs `IIntegrationEventHandler`: same-shaped `Handle(...)` method, zero shared meaning, no common parent interface.

---

## Q: After a domain event handler finishes, how exactly does the integration event handler get called to send an email? What is the exact implementation, and what is the request-pipeline flow?

**Reframed:** Trace the precise code path from "aggregate raises an event inside a command handler" to "an email is actually sent" — including which classes/methods are involved, what runs synchronously vs asynchronously, and whether they're really "chained" together.

**Answer (see full trace in chat, condensed here):**

**The key fact: they are NOT chained by a function call.** The domain handler and the integration handler are connected only through a database row (the outbox), read at two different points in time, in two different execution contexts.

**Phase A — same HTTP request, synchronous:**
1. `TransactionCommandPipeline` opens a SQL transaction on the main business `DbContext`.
2. `DomainEventsPipeline` calls the command handler → aggregate raises event (in-memory only, `Raise(...)`).
3. `ProcessDomainEventsInBatches` collects the event, dispatches it via `IDomainEventDispatcher.PublishAsync<TEvent>` → resolves and runs all `IDomainEventHandler<TEvent>` implementations **synchronously, inside the open transaction** (e.g. `Assets.WorkOrderCompletedEventHandler`, `Technicians.WorkOrderCompletedEventHandler`).
4. The same event is also appended to a plain in-memory list — no integration handler runs yet.
5. `WriteIntegrationEventsToOutbox` serializes each event to JSON and writes an `OutboxMessage` row via `EfCoreOutboxStore.AddAsync`, which calls `SaveChangesAsync()` on a **separate `OutboxDbContext`**.
   - **Confirmed issue:** `OutboxDbContext` is registered independently (`AddDbContext<OutboxDbContext>` with its own connection string) and is never enrolled in the main transaction (`EfUnitOfWork.BeginTransactionAsync` only opens a transaction on the main `DbContext`). So this outbox write commits immediately and independently — if the main transaction later rolls back, the outbox row survives anyway. Logged as a confirmed bug in [bugs-found.md](bugs-found.md).
6. Back in `TransactionCommandPipeline`: main `SaveChangesAsync()` commits business changes, then `transaction.CommitAsync()`. HTTP response returns. **No email has been sent yet.**

**Phase B — separate, later, different thread:**
1. `OutboxProcessor` (a `BackgroundService`, running independently since app startup) wakes every 5 seconds.
2. Claims an unprocessed row with `WITH (UPDLOCK, ROWLOCK, READPAST)`, deserializes the JSON payload back into the real event type via `Type.GetType(message.EventType)` + `JsonSerializer.Deserialize`.
3. Calls `IIntegrationEventDispatcher.PublishAsync(object)` → builds `IntegrationEventHandlerWrapper<TEvent>` via `MakeGenericType` + `Activator.CreateInstance` (reflection needed here only because the compile-time type was erased down to `object` by the JSON round-trip — a different problem than attribute-based behavior lookup).
4. The wrapper resolves `IEnumerable<IIntegrationEventHandler<TEvent>>` from DI (`EmailWorkOrderCompletedHandler`) and calls `Handle(...)` — **this is the exact line where the email is sent.**
5. On success, the outbox row is marked `ProcessedAt = now`. On failure, retry count increments; after `MaxRetries`, moves to dead letter.

*(Full step-by-step with exact code snippets is in the chat transcript from 2026-07-18; ask to have it re-expanded if needed.)*

---

## Q: Explain fully — parameters, inheritance, exact syntax, generics, and return types — for:
```csharp
public interface IDomainEventHandler<in TEvent> where TEvent : IDomainEvent
{
    Task Handle(TEvent domainEvent, CancellationToken cancellationToken = default);
}
public interface IIntegrationEventHandler<in TEvent>
{
    Task Handle(TEvent @event, CancellationToken cancellationToken = default);
}
```
What is `TEvent`? Is `Task` the same kind of thing as declaring `int` or `void`? Do either interface inherit from anything?

**Answer:**

- **`public interface X`** — declares a contract with no implementation; `public` = visible outside this assembly.
- **`<in TEvent>`** — `TEvent` is a **generic type parameter**: a placeholder substituted with a real type later (e.g. `IDomainEventHandler<WorkOrderCompletedEvent>`). Lets one interface serve every event type without duplicating a new interface per event.
- **`in`** — contravariance annotation; restricts `TEvent` to input positions only, enforced by the compiler. Gives flexibility for a handler of a more general type to stand in for a more specific one in some assignment contexts — doesn't affect the DI resolution flow already traced.
- **`where TEvent : IDomainEvent`** — a **generic constraint**: only types implementing `IDomainEvent` are legal for `TEvent` on this interface. `IDomainEventHandler<string>` would not compile. This is what makes the "compile-time safety" claim literal, not figurative.
- **`Task Handle(TEvent domainEvent, CancellationToken cancellationToken = default);`**
  - **`Task`** — grammatically occupies the same "return type slot" as `int` or `void` would — the compiler doesn't treat it as a special keyword. Semantically very different: `void` = no result, no way to observe async completion/errors from outside; `int` = an immediate synchronous value; `Task` = an object representing asynchronous work in progress — `await` it to know when it's done or to observe an exception it captured. `Task` (non-generic) carries no result value, only "done" or "failed." `Task<TResult>` would be used if a value also needed to come back. `Handle` returns `Task` because handling an event is an action, not a value-producing computation.
  - **`TEvent domainEvent`** — first parameter, typed as the generic placeholder.
  - **`CancellationToken cancellationToken = default`** — second parameter; `= default` makes it optional, defaulting to a token that never signals cancellation.
  - Ends in `;` not `{ }` — no body, since this is just the interface's signature declaration.
- **`IIntegrationEventHandler<in TEvent>`** — same shape, no `where` constraint (any type is legal), and the parameter is named `@event` — `event` alone is a reserved C# keyword (unrelated observer-pattern feature: `public event EventHandler Foo;`); `@` lets it be used as a plain identifier anyway. Pure naming coincidence with "domain event" as a DDD concept.
- **Inheritance:** neither interface has a `: SomethingElse` clause. Both are standalone, top-level contracts with no shared parent — confirmed by reading the actual files, not assumed.
