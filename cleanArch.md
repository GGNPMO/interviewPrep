# CQRS and Event-Driven Design: Interview Questions and Detailed Answers

A study guide for senior developer and architect interviews, with a .NET (C#, ASP.NET Core, EF Core) flavor. Every answer is written the way you would explain it out loud: definition, why it matters, how it works, trade-offs, and a concrete example.

> **Notes**
> - Code samples target .NET 8+ and are illustrative sketches, not production-ready.
> - Library details change between versions. MediatR, MassTransit and similar libraries have moved toward commercial licensing in recent major versions, so verify current licensing and APIs before standardizing on them. Alternatives exist (Wolverine, NServiceBus, CAP, or a small hand-rolled dispatcher).
> - Tool names and versions evolve. Treat them as examples of a category, not as the only choice.

---

## How to answer in an interview

Use this shape for almost every design question:

1. **Definition:** one or two sentences in plain words.
2. **Why it exists:** the problem it solves.
3. **How it works:** a small diagram or flow in words.
4. **Trade-offs:** what it costs (complexity, consistency, operations).
5. **Example:** a real system you built or a realistic domain (orders, payments, claims).
6. **When not to use it:** this signals seniority more than anything else.

---

## Table of contents

**Part A: CQRS**
1. What is CQRS and what problem does it solve?
2. CQS vs CQRS
3. What are the benefits of CQRS?
4. When should you NOT use CQRS?
5. Command vs query vs event
6. Levels (flavors) of CQRS
7. Does CQRS require separate databases, event sourcing or microservices?
8. How is the read side kept in sync with the write side?
9. How do you handle eventual consistency in the UI?
10. Where do validation and authorization live?
11. Should a command return data?
12. How do you implement CQRS in .NET?
13. How do CQRS and DDD aggregates fit together?
14. How do you handle concurrency?
15. How do you make commands idempotent?
16. How do you design read models and projections?
17. How do you test a CQRS system?
18. Common CQRS mistakes
19. How do you migrate a CRUD monolith to CQRS incrementally?

**Part B: Event sourcing**

20. What is event sourcing and how does it differ from CRUD?
21. CQRS vs event sourcing: how are they related?
22. What are snapshots and why use them?
23. What are projections and how do you rebuild them?
24. How do you version events?
25. Pros, cons and when not to use event sourcing
26. How do you handle GDPR and deletion with an immutable log?

**Part C: Event-driven architecture**

27. What is event-driven architecture, and what are its benefits and drawbacks?
28. The different meanings of "event-driven"
29. Domain events vs integration events
30. Thin events vs fat events (event-carried state transfer)
31. Commands vs events as messages
32. Choreography vs orchestration
33. The Saga pattern
34. The dual-write problem and the Outbox pattern
35. The Inbox pattern and idempotent consumers
36. Delivery guarantees and "exactly-once"
37. Message ordering
38. Kafka vs RabbitMQ vs Azure Service Bus
39. Queues vs topics and competing consumers
40. Retries, dead-letter queues and poison messages
41. Schema evolution and contract versioning
42. Why not distributed transactions (2PC)?
43. Observability in event-driven systems
44. Scaling consumers and backpressure
45. Testing event-driven systems
46. Security considerations
47. Synchronous vs asynchronous communication: how do you choose?
48. Event-driven anti-patterns

**Part D: Design scenarios**

49. Design an e-commerce order flow
50. Design an insurance claim processing system
51. Clarifying questions to ask before designing

**Part E: Cheat sheet**

---

# Part A: CQRS

## Q1. What is CQRS and what problem does it solve?

**CQRS (Command Query Responsibility Segregation)** is a pattern where you use *different models* for changing state (commands) and for reading state (queries), instead of one model that does both.

**The problem.** In a classic CRUD design, one entity model and one database schema serve every purpose. Over time the pull in opposite directions grows:

- Writes need **normalized** data, strict validation, business rules, transactions and consistency.
- Reads need **denormalized, shaped, fast** data: joins across many tables, aggregates, filters, sorting, paging, search, and different shapes for different screens.

Forcing both into one model leads to bloated entities, heavy ORM queries, lazy-loading surprises, and domain objects polluted with properties only the UI needs. Read and write workloads also scale differently. Many systems read 10 to 100 times more than they write, but a single model forces you to scale and optimize both together.

**How CQRS helps.**

- The **write side** (command model) is small, focused on invariants and behavior, often an aggregate in a DDD sense.
- The **read side** (query model) is built purely for reading: flat DTOs, SQL views, a denormalized table, a search index, or a cache.
- Each side can use its own technology, schema, scaling strategy and team ownership.

**Example.** An order screen shows customer name, order lines, product images, shipping status and totals. On the read side that is one flat query or a precomputed document. On the write side, "add a line to an order" only needs the order aggregate with its business rules (max items, credit limit), not the customer profile, images or shipping data.

**Trade-off.** More moving parts, more code, and often eventual consistency between the sides. It is worth it only when the read/write mismatch is real.

---

## Q2. What is the difference between CQS and CQRS?

**CQS (Command-Query Separation)** is a *method-level* principle from Bertrand Meyer: a method should either **change state** (command, returns nothing) or **return data** (query, no side effects), but not both. It is about how you design individual methods or operations.

**CQRS** is an *architectural* pattern from Greg Young that takes the idea further: separate the **models** (and often the code paths, stores and services) for commands and queries.

| | CQS | CQRS |
|---|---|---|
| Level | Method / class | Architecture / system |
| What is separated | Behavior of a method | Whole models, handlers, often data stores |
| Needs separate data store | No | Optional |
| Typical example | `Stack.Pop()` violates CQS (changes state and returns value) | Separate `PlaceOrderHandler` and `GetOrderSummaryHandler` with different models |

Interview tip: say that CQS is the idea, CQRS is CQS scaled up to the architecture. Also note that pragmatic CQRS often relaxes pure CQS, for example a command handler returning the new ID.

---

## Q3. What are the benefits of CQRS?

1. **Independent optimization.** Write model optimized for consistency and invariants. Read model optimized for speed and shape (denormalized, indexed, cached).
2. **Independent scaling.** Scale read replicas, caches or search clusters without scaling the write database.
3. **Simpler models.** Domain model contains only behavior needed for commands. No need for DTO-shaped properties or lazy-load hacks. Queries return DTOs directly, skipping the domain model and ORM overhead.
4. **Better security separation.** You can grant read-only database credentials to the query side and apply different authorization per operation.
5. **Team and code organization.** Features map to vertical slices (one command or query per file/folder), which reduces merge conflicts and makes code easier to find.
6. **Flexibility in storage.** The read side can be SQL view, Elasticsearch, Redis, Cosmos DB or a materialized view, whichever fits the query.
7. **Natural fit with events.** Commands produce events, and events feed read models. This makes audit trails and integrations easier.
8. **Performance.** Read queries avoid ORM tracking, joins through the domain graph, and mapping overhead (for example Dapper or raw SQL returning DTOs).

Be honest in the interview: these benefits only matter above a certain complexity. For a simple admin CRUD app, most of them are not worth the cost.

---

## Q4. When should you NOT use CQRS?

Avoid it when:

- The domain is **simple CRUD** with few business rules.
- The team is **small or inexperienced** with the pattern, and the operational burden of eventual consistency is not justified.
- Read and write models are **nearly identical** and the load is modest.
- You need **strong immediate consistency everywhere** and cannot tolerate even short staleness, and you are not prepared to engineer around it.
- The system is early-stage and requirements are still changing rapidly. Premature separation adds friction.

Greg Young and Martin Fowler both warn that CQRS is a significant mental leap and that many systems do not benefit. A good rule: apply CQRS to the **parts** of a system with a real complexity or scaling problem (for example, a reporting-heavy dashboard or a complex ordering domain), not to the whole system by default.

---

## Q5. What is the difference between a command, a query and an event?

| | Command | Query | Event |
|---|---|---|---|
| Intent | "Do this" (request to change state) | "Tell me this" (request for data) | "This happened" (fact) |
| Tense | Imperative: `PlaceOrder`, `CancelClaim` | Question: `GetOrderById` | Past tense: `OrderPlaced`, `ClaimApproved` |
| Can be rejected? | Yes (validation, rules) | No (it just returns data) | No (it already happened) |
| Side effects | Yes | None | It is the record of side effects |
| Recipients | Exactly one handler | One handler | Zero, one or many subscribers |
| Return | Nothing or minimal (ID, status) | Data | Nothing |

Why the distinction matters: a command expresses **intent** and can fail, so the sender cares about the outcome. An event is a **fact**, so publishers should not care who consumes it. Mixing these up (for example publishing an "event" that is really a command to a specific service) creates hidden coupling.

---

## Q6. What are the levels or flavors of CQRS?

CQRS is a spectrum, not a single design:

1. **Logical separation, same model and database.** Separate command handlers and query handlers in code, but both use the same entities and DB. This gives cleaner code and is the cheapest step.
2. **Different models, same database.** Commands go through EF Core and the domain model. Queries use Dapper, SQL views or projections directly into DTOs. Same database, different access paths. Often the best starting point.
3. **Separate read store, updated asynchronously.** Writes go to the primary database. Events or change data capture update a separate read store (for example Elasticsearch, Redis, Cosmos DB, a reporting database). Eventual consistency appears here.
4. **CQRS plus event sourcing.** The write side stores events instead of current state. Read models are projections built from the event stream.

Interview answer: "I choose the lowest level that solves the actual problem. Most teams get 80 percent of the value at level 1 or 2 without eventual consistency."

---

## Q7. Does CQRS require separate databases, event sourcing or microservices?

**No to all three.**

- **Separate databases:** optional. Separate models can read the same database. Separate stores are used when scale or query technology demands it.
- **Event sourcing:** optional and independent. CQRS works with a normal state-based write store. Event sourcing works without CQRS (although projections for reading nearly always lead you to CQRS-like separation).
- **Microservices:** optional. You can use CQRS inside a monolith (a modular monolith) or inside a single microservice.

Common confusion: people describe CQRS + event sourcing + Kafka + microservices as "CQRS". These are separate patterns that happen to combine well. Say so in the interview.

---

## Q8. How is the read side kept in sync with the write side?

Options, from simplest to most decoupled:

1. **Same database, same transaction.** Read model is a view or an indexed table updated in the same transaction. Strong consistency, but couples the sides.
2. **Synchronous projection in-process.** After the command commits, a handler updates the read tables. Easy, but if it fails you need to reconcile.
3. **Asynchronous via domain/integration events.** The command side publishes events (through the **Outbox pattern**, see Q34). A projection handler consumes them and updates the read store. Eventual consistency, decoupled, scalable.
4. **Change Data Capture (CDC).** A tool such as Debezium reads the database transaction log and streams changes to a broker, and projections consume them. No application changes to publish, but the events reflect table changes rather than business intent.
5. **Event sourcing projections.** Projections subscribe to the event store and maintain read models. They can be rebuilt by replaying history.

Key engineering requirements for any asynchronous approach:

- **Idempotent projections** (the same event twice must not corrupt the read model).
- **Ordering per aggregate** (or tolerance for out-of-order events using version numbers).
- **Checkpointing** (store the last processed position so you can resume).
- **Rebuild ability** (be able to drop and recreate the read model from source).
- **Monitoring of lag** (how far behind is the read model).

---

## Q9. How do you handle eventual consistency in the UI?

Eventual consistency means that right after a command succeeds, a query might return stale data. Strategies:

1. **Return what the client needs from the command.** After `POST /orders`, return the order ID (and maybe the created resource). The UI shows that data locally without querying.
2. **Optimistic UI.** Update the screen immediately assuming success, and reconcile if the server reports failure.
3. **Asynchronous acknowledgement.** Return `202 Accepted` with a `Location` header pointing to a status resource. The client polls, or the server pushes completion through SignalR, WebSockets or server-sent events.
4. **Read-your-own-writes via version/position.** The command returns a version or event position. The client asks the query side for data at least that version. The query side waits briefly (with a timeout) until its projection checkpoint passes that version.
5. **Route the user's own immediate read to the write store.** For a short window after a write, read from the primary instead of the lag-prone replica or read store.
6. **Design the UX around it.** Show a "processing" state, redirect to a list page that tolerates lag, or use a toast such as "Your claim has been submitted."
7. **Set expectations in requirements.** Agree with the business on acceptable staleness (for example under one second, or under five seconds). Many screens tolerate it. Money transfers and inventory checks at the point of purchase usually must be consistent and should be validated against the write model.

---

## Q10. Where do validation and authorization live in CQRS?

Break validation into layers:

1. **Input/syntactic validation** (required fields, formats, lengths): in the command validator, typically **FluentValidation** in a pipeline behavior that runs before the handler. Fast failure, returns `400`.
2. **Business invariants** (order total cannot exceed credit limit, cannot cancel a shipped order): in the **aggregate** (domain model). This is the source of truth, because it protects consistency.
3. **Cross-aggregate or uniqueness rules** (email must be unique): enforce with a **database unique constraint** as the final guard, plus an application-level check for friendlier errors. Do not rely on the read model for this check, because it can be stale.

Important rule: **never validate invariants against the read model.** It might be behind. Use the write model or the database.

**Authorization:** apply at the boundary. Typical approach: authentication in middleware, coarse authorization via policies on the endpoint, and fine-grained (resource-based) authorization in a pipeline behavior or inside the handler (for example, "this user may only cancel their own claim"). Queries need authorization too, including row-level or tenant filtering. A tenant filter should be applied centrally so no query can forget it.

---

## Q11. Should a command return data?

Purists say no: a command returns nothing (or just success/failure), and the client issues a query to read the result. This keeps strict CQS and works well with asynchronous processing.

Pragmatic teams allow commands to return **minimal information**:

- The generated **ID** (so the client can navigate to the resource).
- A **status or result object** (success, validation errors, concurrency token).
- A **version/ETag** for read-your-writes (see Q9).

What commands should **not** return is a full read model, because that couples the command side to the query shape and duplicates query logic.

If the command is processed asynchronously, it returns `202 Accepted` with a tracking ID. Be consistent across your API and document the convention.

---

## Q12. How do you implement CQRS in .NET?

A common stack: **ASP.NET Core Minimal APIs or controllers**, a mediator (MediatR or an equivalent), **FluentValidation**, **EF Core** for writes and **Dapper** (or EF Core with `AsNoTracking` and projections) for reads.

### Commands and handlers (write side)

```csharp
public sealed record PlaceOrderCommand(
    Guid CustomerId,
    IReadOnlyList<OrderLineDto> Lines,
    Guid IdempotencyKey) : IRequest<Guid>;

public sealed class PlaceOrderHandler(AppDbContext db) : IRequestHandler<PlaceOrderCommand, Guid>
{
    public async Task<Guid> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Place(
            cmd.CustomerId,
            cmd.Lines.Select(l => new OrderLine(l.ProductId, l.Quantity, l.UnitPrice)));

        db.Orders.Add(order);          // aggregate enforces invariants
        await db.SaveChangesAsync(ct); // one transaction per command
        return order.Id;
    }
}
```

### Queries and handlers (read side)

```csharp
public sealed record GetOrderSummaryQuery(Guid OrderId) : IRequest<OrderSummaryDto?>;

public sealed class GetOrderSummaryHandler(IDbConnectionFactory factory)
    : IRequestHandler<GetOrderSummaryQuery, OrderSummaryDto?>
{
    public async Task<OrderSummaryDto?> Handle(GetOrderSummaryQuery q, CancellationToken ct)
    {
        using var conn = factory.Create();
        const string sql = """
            SELECT o.Id, o.Status, o.Total, c.Name AS CustomerName
            FROM Orders o
            JOIN Customers c ON c.Id = o.CustomerId
            WHERE o.Id = @OrderId
            """;
        return await conn.QuerySingleOrDefaultAsync<OrderSummaryDto>(
            new CommandDefinition(sql, new { q.OrderId }, cancellationToken: ct));
    }
}
```

### Cross-cutting behavior with a pipeline

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse> where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var results = await Task.WhenAll(validators.Select(v => v.ValidateAsync(request, ct)));
        var failures = results.SelectMany(r => r.Errors).Where(f => f is not null).ToList();
        if (failures.Count > 0) throw new ValidationException(failures);
        return await next();
    }
}
```

(The exact `RequestHandlerDelegate` signature differs between library versions.)

Other behaviors worth adding: logging, authorization, transaction handling, idempotency check, performance timing, and caching for queries.

### Endpoint

```csharp
app.MapPost("/orders", async (PlaceOrderCommand cmd, ISender sender) =>
{
    var id = await sender.Send(cmd);
    return Results.Created($"/orders/{id}", new { id });
});

app.MapGet("/orders/{id:guid}", async (Guid id, ISender sender) =>
    await sender.Send(new GetOrderSummaryQuery(id)) is { } dto ? Results.Ok(dto) : Results.NotFound());
```

**Folder structure:** prefer **vertical slices** (`Features/Orders/PlaceOrder/{Command, Handler, Validator, Endpoint}.cs`) over horizontal layers, so each feature's code lives together.

---

## Q13. How do CQRS and DDD aggregates fit together?

DDD aggregates define **consistency boundaries** on the write side. Each command targets **one aggregate instance**, loads it, calls a method that enforces invariants, and saves it in one transaction. That maps naturally to a command handler.

- The **aggregate** lives on the command side. It exposes behavior (`order.AddLine(...)`, `claim.Approve(...)`), not setters.
- **Repositories** load and save whole aggregates for commands. They are not used for queries.
- The **query side** ignores aggregates. It reads projections shaped for the UI.
- Aggregates raise **domain events** (`OrderPlaced`) that handlers use to update read models or trigger other workflows.
- One command should change **one aggregate**. If a business operation needs several, coordinate them with events or a saga (eventual consistency between aggregates), not one giant transaction.

Why this matters: because the write model is only used for behavior, it can stay small and rich. It does not need to expose data for reports or screens.

---

## Q14. How do you handle concurrency?

Two users or processes may modify the same aggregate at the same time. Use **optimistic concurrency**:

- **EF Core:** add a `rowversion` concurrency token. When saving, EF includes the version in the `WHERE` clause. If another change got there first, zero rows are updated and `DbUpdateConcurrencyException` is thrown.

```csharp
public class Order
{
    public Guid Id { get; private set; }
    [Timestamp] public byte[] RowVersion { get; private set; } = default!;
}
```

- **Event-sourced systems:** append events with an **expected version**. The event store rejects the append if the stream's current version differs.
- **API level:** expose the version as an **ETag** and require `If-Match` on updates. Return `409 Conflict` or `412 Precondition Failed` on mismatch.

How to react to a conflict:

1. Reload and **retry automatically** if the command is safe to re-run against the latest state (for example, "add line").
2. Return the conflict to the user to resolve when intent may no longer apply (for example, "edit the same text field").

Use **pessimistic locking** only for rare, short, high-contention cases, since it limits scalability. For hot spots (inventory counters) consider atomic updates (`UPDATE ... SET Stock = Stock - @qty WHERE Stock >= @qty`) rather than load-modify-save.

---

## Q15. How do you make commands idempotent?

Clients retry (timeouts, double clicks, network failure), and brokers may redeliver. A non-idempotent command, such as "charge card", is dangerous when duplicated.

Techniques:

1. **Idempotency key.** The client sends a unique key (header or command field). The server stores processed keys with the result. A repeated key returns the stored result without re-executing.
2. **Natural idempotency.** Prefer operations such as "set status to Approved" over "increment counter". `PUT` semantics and upserts are idempotent by design.
3. **Unique constraints.** Make the database reject duplicates (unique index on the idempotency key or business key).
4. **Deduplication at the broker.** Azure Service Bus offers duplicate detection by `MessageId` within a time window. Useful, but not enough alone.
5. **Aggregate state checks.** If the order is already `Paid`, a second `MarkPaid` is ignored or returns success.

Implementation sketch: a pipeline behavior checks an `IdempotencyRecords` table inside the same transaction as the handler. If the key exists, return the saved response. Otherwise run the handler and store the key and response together.

---

## Q16. How do you design read models and projections?

Principles:

- **Design per use case.** Build one read model per screen, API or report, even if data duplicates. Storage is cheap, joins at read time are not.
- **Denormalize deliberately.** Store customer name inside the order summary row, so the list view needs no join.
- **Choose store by query shape.** Relational tables for filtering and sorting, Elasticsearch/Azure AI Search for text search, Redis for hot lookups, document stores for nested JSON.
- **Make projections idempotent and rebuildable.** Track `LastProcessedEventId` or version per projection.
- **Handle out-of-order and duplicate events.** Compare the event's aggregate version with the stored version and ignore stale ones.
- **Version your read models.** For schema changes, build the new projection side by side (`OrderSummary_v2`), catch up from history, then switch traffic. No downtime.
- **Include only what is needed.** Avoid turning one projection into a copy of the whole write model.

Example projection handler:

```csharp
public async Task Handle(OrderPlaced e, CancellationToken ct)
{
    var exists = await _db.OrderSummaries.AnyAsync(x => x.OrderId == e.OrderId, ct);
    if (exists) return; // idempotent

    _db.OrderSummaries.Add(new OrderSummary
    {
        OrderId = e.OrderId,
        CustomerName = e.CustomerName,
        Total = e.Total,
        Status = "Placed",
        Version = e.Version
    });
    await _db.SaveChangesAsync(ct);
}
```

---

## Q17. How do you test a CQRS system?

- **Unit tests for aggregates.** Pure domain logic, no infrastructure. For event-sourced aggregates use *Given (past events) / When (command) / Then (new events or exception)*.
- **Handler tests.** Test command handlers with an in-memory or containerized database. Test query handlers against a real database, since SQL correctness matters.
- **Validator tests.** FluentValidation rules are cheap to test exhaustively.
- **Projection tests.** Feed a sequence of events and assert the resulting read model. Include duplicates and out-of-order cases.
- **Integration tests.** Use **Testcontainers** for SQL Server, Postgres, RabbitMQ or Kafka with `WebApplicationFactory` to exercise the whole pipeline.
- **Eventual consistency tests.** Use polling with timeout (not fixed sleeps) to wait for the read model to catch up.
- **Contract tests** for integration events (consumer-driven, for example Pact).
- **Replay tests.** Rebuild a projection from a recorded event stream and compare to the live read model.

---

## Q18. What are the common CQRS mistakes?

1. **Applying it everywhere.** Even to trivial CRUD screens.
2. **Treating it as "two databases".** The core is separate models, not technology.
3. **Leaking domain entities into queries**, or leaking query DTOs into the domain model.
4. **Ignoring eventual consistency.** UI shows stale data and users complain, or code validates invariants against stale read models.
5. **Fat handlers.** Business logic in handlers instead of aggregates (anemic domain model with a mediator on top).
6. **Commands that change several aggregates** in one transaction, defeating the aggregate boundary.
7. **No idempotency**, causing duplicates on retries.
8. **No way to rebuild projections**, so any bug becomes permanent data corruption.
9. **Mediator overuse.** Handlers calling other handlers via the mediator, creating hidden call chains.
10. **No monitoring of projection lag.**
11. **Over-engineering with event sourcing** when only simple CQRS was needed.

---

## Q19. How do you migrate a CRUD monolith to CQRS incrementally?

Use the **strangler** approach, not a rewrite:

1. **Find the pain.** Slow reports, complex screens or contention on a few tables. Pick one bounded area.
2. **Introduce logical separation first.** Create query handlers that read via Dapper or SQL views into DTOs for that area, while commands still use the old service layer. Immediate performance wins, no consistency change.
3. **Add a command layer** for that area: commands and handlers wrapping the existing logic, then move logic into aggregates over time.
4. **Add read models only where needed.** Create a projection table or search index populated by domain events (with Outbox). Keep the old path in parallel until verified.
5. **Dual-run and compare.** Compare results from old and new read paths, then switch traffic gradually (feature flag).
6. **Repeat for the next area.** Leave low-value CRUD areas alone.

Risks to manage: data drift between old and new paths, partial rollouts, and team learning curve. Mitigate with automated comparison, feature flags, and clear ownership.

---

# Part B: Event sourcing

## Q20. What is event sourcing and how does it differ from CRUD?

**Event sourcing** stores the state of an entity as an **append-only sequence of events** instead of overwriting the current state. The current state is derived by replaying events in order.

| | CRUD / state-based | Event sourcing |
|---|---|---|
| What is stored | Current state (latest row) | Every change as an immutable event |
| Update | Overwrite | Append a new event |
| History | Lost unless you add audit tables | Complete and built in |
| "Why did it change?" | Unknown | Recorded in the event (`ClaimRejected` with reason) |
| Read current state | Direct | Replay (or snapshot + replay) or read a projection |

**Example.** An account stores `AccountOpened(100)`, `MoneyDeposited(50)`, `MoneyWithdrawn(30)`. Balance is 120, derived by replay. Storing only `Balance = 120` loses how it got there.

Core mechanics:

- Each aggregate has an **event stream** (identified by aggregate ID).
- To handle a command: load events, rebuild the aggregate, run the behavior, get **new events**, append them with an **expected version**.
- Events are **facts in the past tense**, are **immutable**, and are never edited or deleted (corrections are new compensating events).

Simple aggregate skeleton:

```csharp
public abstract class AggregateRoot
{
    private readonly List<object> _uncommitted = new();
    public Guid Id { get; protected set; }
    public int Version { get; private set; }

    public IReadOnlyList<object> GetUncommittedEvents() => _uncommitted;

    public void Load(IEnumerable<object> history)
    {
        foreach (var e in history) { Apply(e); Version++; }
    }

    protected void Raise(object @event)
    {
        Apply(@event);
        _uncommitted.Add(@event);
    }

    protected abstract void Apply(object @event);
}
```

---

## Q21. CQRS vs event sourcing: how are they related?

They are **independent patterns** that complement each other.

- **CQRS** separates write and read models.
- **Event sourcing** changes how the write side persists state (as events).

Why they pair well: an event log is great for writing (append-only, no joins) but awkward for querying ("all orders over 1000 in region X"). Projections solve this by building read models from the events, which is exactly what CQRS's read side wants.

You can use:

- CQRS **without** event sourcing (most common).
- Event sourcing **without** CQRS (rare, because querying the raw log is hard, although you can read current state from a snapshot or a single stream).
- **Both** together (common for audit-heavy or highly temporal domains).

---

## Q22. What are snapshots and why use them?

A long-lived aggregate may accumulate thousands of events. Replaying all of them on every command is slow.

A **snapshot** is a saved serialized copy of the aggregate state at a specific version. To load: fetch the latest snapshot, then replay only events **after** it.

Practical points:

- Take snapshots **every N events** (for example every 100) or by time.
- They are a **performance optimization only**. The event stream remains the source of truth. You can delete and regenerate snapshots.
- Snapshot schema must be versioned. If the aggregate code changes, an old snapshot may not deserialize, so be ready to fall back to full replay.
- **Do not add snapshots prematurely.** Most aggregates have short lifespans (an order has tens of events), so measure first. Design streams to be short instead (for example one stream per claim, not one per customer forever).

---

## Q23. What are projections and how do you rebuild them?

A **projection** subscribes to events and builds a read-optimized view (SQL table, document, search index).

Types:

- **Inline (synchronous):** updated in the same transaction as the event append. Strong consistency but couples write latency to projection work.
- **Asynchronous:** a background subscriber processes events and updates the view. Eventually consistent, scalable, and the common default.
- **Live aggregation / on-demand:** computed at query time from a stream (for a single entity).

**Rebuilding:** because the events are the source of truth, you can drop a projection and replay all events (or all events of a type) to recreate it. Useful for bug fixes, new reports, or schema changes.

Rebuild best practices:

- Build the new version **side by side** and switch over when caught up (blue/green projections).
- Run catch-up with **batching and parallelism** for performance.
- Keep handlers **deterministic and side-effect free** (no emails or external calls during replay).
- Store a **checkpoint** so a crash can resume.

---

## Q24. How do you version events?

Events live forever, and your code evolves. A 2024 event must still be readable in 2030.

Approaches:

1. **Additive, backward-compatible changes.** Add optional fields with defaults. Never remove or repurpose fields.
2. **Upcasting.** When reading an old event, transform it in memory to the latest shape (`OrderPlacedV1` to `OrderPlacedV2`) before applying. Stored data is unchanged.
3. **New event types for semantic change.** If meaning changes, introduce `OrderPlacedV2` or a differently named event rather than changing the old one.
4. **Weak schema / tolerant reader.** Use JSON with tolerant deserialization.
5. **Copy-and-transform (migration).** Rarely, write a new stream with transformed events. This is heavy and breaks the immutability principle, so use it only when necessary.
6. **Schema registry** (Avro/Protobuf/JSON Schema) to enforce compatibility rules in CI.

Include an event **type name and schema version** in metadata. Treat events as a public contract, especially ones that leave the service as integration events.

---

## Q25. What are the pros, cons and non-use cases of event sourcing?

**Pros**

- Complete **audit trail** and full history for free.
- **Temporal queries** ("what did the claim look like on 1 March?").
- Natural **integration** through events.
- **Replay and rebuild** read models, fix bugs by reprocessing.
- Debugging by replaying production sequences.
- Good fit for domains where history is the business (finance, insurance claims, logistics).

**Cons**

- **Steep learning curve** and mindset shift.
- **Event versioning** is a permanent responsibility.
- **Querying** requires projections, so CQRS comes with it.
- **Eventual consistency** between write and read sides.
- **Storage growth** and replay time (mitigated by snapshots, archiving).
- **Data deletion** (GDPR) needs special handling (Q26).
- **Operational and tooling maturity** is lower than relational databases.
- Mistakes in event design are costly to undo.

**Avoid when:** the domain is CRUD-like, there is no real value in history, the team is inexperienced, or reporting needs dominate with ad-hoc queries. A cheaper alternative is an **audit table or temporal tables** with a normal state store.

Tools: EventStoreDB (recently rebranded as Kurrent), Marten (on PostgreSQL), SQL-based custom stores, Axon (JVM). Verify current product status when choosing.

---

## Q26. How do you handle GDPR and deletion in an immutable event log?

The right to erasure conflicts with immutable events. Practical strategies:

1. **Keep personal data out of events.** Store a reference (customer ID) and keep personal data in a separate, deletable store. The best option when possible.
2. **Crypto-shredding.** Encrypt personal fields with a **per-subject key** stored in a key vault. To "delete" the person, destroy the key. Events remain but the personal data is unreadable.
3. **Tombstone/anonymization events.** Append an event such as `CustomerAnonymized`, and projections purge or mask data. Underlying old events still contain data unless you also rewrite them.
4. **Stream rewrite/archive.** As a last resort, rewrite streams with redacted data. This violates the immutability principle and needs strict controls.

Also consider retention: define how long events must be kept, and archive older streams to cheaper storage. Consult legal and compliance for jurisdiction-specific rules.

---

# Part C: Event-driven architecture

## Q27. What is event-driven architecture, and what are its benefits and drawbacks?

**Event-driven architecture (EDA)** is a style where components communicate by **producing and reacting to events** (facts about something that happened) rather than calling each other directly. A producer publishes an event to a broker without knowing who consumes it. Consumers react independently.

**Benefits**

- **Loose coupling.** Producers do not know consumers. Add a new subscriber (for example analytics) without touching the producer.
- **Scalability.** Consumers scale horizontally and process at their own pace.
- **Resilience.** If a consumer is down, events queue up and are processed later (temporal decoupling).
- **Extensibility.** New behaviors are new subscribers.
- **Real-time reactions** and streaming.
- **Natural audit** of what happened.

**Drawbacks**

- **Eventual consistency** is the default, which complicates UX and logic.
- **Harder to reason about and debug.** Flow is spread across services (no single call stack). Needs tracing.
- **Duplicates, retries and out-of-order messages** must be handled everywhere.
- **Operational complexity.** Broker operations, monitoring, schema governance.
- **Hidden coupling** through event contracts and implicit dependencies.
- **Testing** is more involved.

Use EDA where decoupling and scale matter. Do not use it for simple request-response needs ("get me the price now"), where a synchronous call is simpler.

---

## Q28. What are the different meanings of "event-driven"?

Martin Fowler pointed out that people use the term for four different patterns. Naming which one you mean avoids confusion:

1. **Event notification.** A service publishes a small event ("OrderPlaced, id 42"). Interested services react and may call back to fetch details. Low coupling, but chatty callbacks and hard-to-see flows.
2. **Event-carried state transfer.** The event contains enough data for consumers to update their own local copy, so they do not need to call back. Fewer runtime dependencies, but data duplication and staleness.
3. **Event sourcing.** State is stored as events (Part B).
4. **CQRS** using events to update read models.

In an interview, ask: "Which kind of event-driven design do we mean here?" That shows maturity.

---

## Q29. What is the difference between domain events and integration events?

| | Domain event | Integration event |
|---|---|---|
| Scope | Inside one bounded context / service | Across services / bounded contexts |
| Raised by | Aggregates | Application layer after a domain event or command |
| Delivery | Usually in-process, same transaction or immediately after | Through a message broker, asynchronous |
| Contract | Internal, can change freely | Public, versioned, backward compatible |
| Content | Rich domain details | Minimal, stable, intentionally designed |
| Failure handling | Part of the same unit of work | Retries, DLQ, outbox, idempotency |

**Why separate them.** If you publish internal domain events directly, any refactor of the domain model breaks other teams. Map domain events to **integration events** deliberately so the external contract stays stable.

**Dispatching domain events in EF Core.** Common approach: aggregates collect events in a list. Override `SaveChangesAsync` to gather them from tracked entities and dispatch.

- **Before commit** (dispatch, then save): handlers run in the same transaction, so side effects are atomic with the change. Good for in-process consistency, but risky for slow or external work.
- **After commit** (save, then dispatch): safe for side effects, but if the process crashes between commit and dispatch, events are lost.

For anything leaving the service, write the integration event to an **outbox** in the same transaction (see Q34) so it is never lost.

---

## Q30. Thin events vs fat events: what is event-carried state transfer?

**Thin event (notification):** contains only identifiers and minimal info, for example `OrderPlaced { OrderId }`. Consumers call the producer's API to fetch details.

- Pros: small, stable contract, always fresh data on fetch.
- Cons: **runtime coupling** (producer must be up and scalable for callbacks), chatty traffic, possible thundering herd after a burst of events.

**Fat event (event-carried state transfer):** contains the data consumers need, for example `OrderPlaced { OrderId, CustomerId, Lines[], Total, ShippingAddress }`.

- Pros: consumers are **autonomous** and need no callbacks. Better resilience and performance.
- Cons: larger payloads, **data duplication**, larger contract surface (harder to evolve), risk of leaking sensitive data, consumers may hold stale copies.

**Guideline.** Include what most consumers legitimately need (a "right-sized" event). Use thin events when data is large, sensitive or changes very often. Use fat events when consumers must work while the producer is unavailable. Often a hybrid works: key facts in the event plus an ID for fetching extras.

---

## Q31. Commands vs events as messages: why does it matter?

Messages on a broker come in kinds with different semantics:

- **Command message:** "Do X" directed at **one specific handler/owner**. Example: `ReserveInventory`. The sender knows the receiver's capability and expects it to act (or fail). Sent to a **queue**.
- **Event message:** "X happened". Published to **many unknown subscribers**. Example: `InventoryReserved`. Sent to a **topic/exchange**.

Why it matters:

- **Naming:** commands are imperative, events are past tense.
- **Ownership:** the owner of the capability owns the command contract. The publisher owns the event contract.
- **Coupling:** commands create dependency on the receiver (the sender knows who). Events invert it.
- **Failure semantics:** a command can be rejected (and the sender cares), an event cannot.

Anti-pattern: an "event" like `SendWelcomeEmail` is really a command disguised as an event. Publish `UserRegistered` instead and let the email service decide what to do.

---

## Q32. Choreography vs orchestration

Two ways to coordinate a multi-service business process:

**Choreography.** No central controller. Each service reacts to events and emits its own. Flow: `OrderPlaced` leads to Inventory reserves and emits `InventoryReserved`, then Payment charges and emits `PaymentCompleted`, then Shipping creates a shipment.

- Pros: very loose coupling, no single point of failure, easy to add participants.
- Cons: **the process is implicit and scattered**, hard to see or change end to end, risk of cyclic or spaghetti dependencies, hard to implement timeouts and compensation clearly.

**Orchestration.** A central **orchestrator** (saga orchestrator or workflow engine) tells each participant what to do through commands and tracks state.

- Pros: process is **explicit, visible and testable**, easier compensation, timeouts and monitoring.
- Cons: the orchestrator is a coupling point and must be designed for availability, and risk of a "god" orchestrator holding business logic that belongs in services.

**Rule of thumb.** Choreography for simple, short flows with few participants. Orchestration for long-running, multi-step processes with compensation, timeouts and business visibility. Tools: MassTransit state machines, NServiceBus sagas, Azure Durable Functions, Temporal, Camunda, Dapr workflows.

---

## Q33. What is the Saga pattern?

A **saga** manages a business transaction that spans multiple services (each with its own database) as a **sequence of local transactions**, each publishing an event or message to trigger the next. If a step fails, the saga runs **compensating actions** to undo the previous steps' effects.

**Example: place order**

1. Order service creates order (Pending).
2. Inventory reserves stock.
3. Payment charges the card.
4. Shipping creates a shipment.
5. Order is marked Confirmed.

If Payment fails at step 3, compensations: release the inventory reservation, mark the order Cancelled (and notify the customer).

**Key points**

- **Compensation is not rollback.** There is no isolation, and other transactions may have seen intermediate state. A compensation is a business-level undo (refund, release, cancel), which must itself be idempotent and retriable.
- **Steps classification:** *compensable* (can be undone), *pivot* (the point of no return), *retriable* (must eventually succeed after the pivot).
- **Lack of isolation** means you must handle anomalies (for example, semantic locks like a `Pending` state, commutative updates, re-reading values before acting).
- **Persistence:** saga state is stored durably so it survives restarts.
- **Timeouts:** define what happens if a reply never arrives (for example, cancel after 15 minutes).
- **Idempotency** on every participant, since messages are delivered at least once.

Illustrative orchestrated saga sketch (MassTransit-style):

```csharp
public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = default!;
    public Guid OrderId { get; set; }
}

public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State AwaitingPayment { get; private set; } = default!;
    public State Failed { get; private set; } = default!;
    public Event<OrderSubmitted> OrderSubmitted { get; private set; } = default!;
    public Event<PaymentCompleted> PaymentCompleted { get; private set; } = default!;
    public Event<PaymentFailed> PaymentFailed { get; private set; } = default!;

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Initially(
            When(OrderSubmitted)
                .Then(ctx => ctx.Saga.OrderId = ctx.Message.OrderId)
                .Publish(ctx => new ReserveInventory(ctx.Saga.OrderId))
                .TransitionTo(AwaitingPayment));

        During(AwaitingPayment,
            When(PaymentCompleted)
                .Publish(ctx => new CreateShipment(ctx.Saga.OrderId))
                .Finalize(),
            When(PaymentFailed)
                .Publish(ctx => new ReleaseInventory(ctx.Saga.OrderId))
                .TransitionTo(Failed));
    }
}
```

(Abbreviated and illustrative. Event correlation configuration is omitted.)

---

## Q34. What is the dual-write problem and how does the Outbox pattern solve it?

**Dual-write problem.** A service must update its database **and** publish an event. These are two separate systems with no shared transaction:

- Save to DB succeeds, publish fails: the rest of the system never learns about the change (lost event).
- Publish first, then DB save fails: the system learned about something that never happened (phantom event).

Retrying or ordering the two calls cannot make this fully safe, because the process can crash between them.

**Outbox pattern.**

1. In **one local database transaction**, write the business change **and** insert the outgoing message into an `OutboxMessages` table.
2. A separate **relay** (background worker, or CDC tool such as Debezium) reads unsent outbox rows and publishes them to the broker.
3. After a successful publish, mark the row as processed (or delete it).

Because step 1 is atomic, either both the change and the message exist, or neither does. The relay gives **at-least-once** delivery, so consumers must be idempotent (Q35).

```sql
CREATE TABLE OutboxMessages (
    Id            UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    Type          NVARCHAR(500)    NOT NULL,
    Payload       NVARCHAR(MAX)    NOT NULL,
    OccurredOnUtc DATETIME2        NOT NULL,
    ProcessedOnUtc DATETIME2       NULL
);
```

```csharp
public sealed class OutboxPublisher(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = scopes.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var bus = scope.ServiceProvider.GetRequiredService<IMessagePublisher>();

            var batch = await db.OutboxMessages
                .Where(m => m.ProcessedOnUtc == null)
                .OrderBy(m => m.OccurredOnUtc)
                .Take(50)
                .ToListAsync(ct);

            foreach (var m in batch)
            {
                await bus.PublishAsync(m.Type, m.Payload, m.Id, ct);
                m.ProcessedOnUtc = DateTime.UtcNow;
            }

            await db.SaveChangesAsync(ct);
            if (batch.Count == 0) await Task.Delay(TimeSpan.FromSeconds(1), ct);
        }
    }
}
```

Production concerns:

- **Multiple relay instances:** use row locking (for example `UPDLOCK, READPAST` in SQL Server) or a leader election, otherwise two relays may publish the same message.
- **Ordering:** relay in insertion order per aggregate if consumers depend on it.
- **Cleanup:** purge old processed rows.
- **Monitoring:** alert on outbox backlog and age of the oldest unsent message.
- **Ready-made options:** MassTransit and NServiceBus have built-in outbox support. CAP and Wolverine also provide it. Debezium offers log-based outbox publishing.

---

## Q35. What is the Inbox pattern, and why must consumers be idempotent?

Because brokers and outbox relays give **at-least-once** delivery, the same message can arrive more than once (redelivery after a timeout, relay crash after publishing but before marking sent, consumer crash before acknowledgment).

A consumer that is not idempotent will double-charge, double-ship or double-count.

**Inbox pattern.** The consumer records processed message IDs:

1. Start a local transaction.
2. Check whether `MessageId` exists in the `InboxMessages` table. If so, acknowledge and return.
3. Perform the business work.
4. Insert the `MessageId` into the inbox **in the same transaction**, then commit.
5. Acknowledge the message.

A **unique index on MessageId** protects against two concurrent deliveries racing.

```csharp
public async Task Consume(ConsumeContext<PaymentCompleted> ctx)
{
    var messageId = ctx.MessageId!.Value;

    if (await db.InboxMessages.AnyAsync(x => x.MessageId == messageId))
        return; // already processed

    var order = await db.Orders.FindAsync(ctx.Message.OrderId);
    order!.MarkPaid(ctx.Message.PaymentId);

    db.InboxMessages.Add(new InboxMessage(messageId, DateTime.UtcNow));
    await db.SaveChangesAsync(); // business change and inbox row commit atomically
}
```

Other idempotency options: make the operation naturally idempotent (set-state rather than increment), use business keys (`PaymentId` unique), or use version checks. Clean old inbox rows after the maximum redelivery window.

---

## Q36. What are the delivery guarantees, and is "exactly-once" real?

| Guarantee | Meaning | Risk |
|---|---|---|
| **At-most-once** | Message delivered zero or one time. No retries. | Message loss |
| **At-least-once** | Message delivered one or more times. Retry until acknowledged. | Duplicates |
| **Exactly-once** | Delivered and processed once | Hard or impossible end to end |

In distributed systems, true end-to-end exactly-once delivery across independent systems is not achievable in general. What systems provide is **exactly-once *processing effect*** by combining **at-least-once delivery + idempotent processing** (or transactional processing within one system).

Notes:

- **Kafka** offers idempotent producers and transactions that give exactly-once semantics **within Kafka** (read-process-write between topics). Side effects outside Kafka (database, email, payment API) still need idempotency.
- **Azure Service Bus** offers duplicate detection on the send side plus peek-lock for at-least-once on the receive side.
- **RabbitMQ** gives at-least-once with acknowledgments and publisher confirms.

Interview answer: "I design for at-least-once delivery and make handlers idempotent. I treat 'exactly-once' claims as scoped to a particular system."

---

## Q37. How do you handle message ordering?

Global ordering is expensive and limits scale. Aim for ordering only where the business needs it, typically **per aggregate or per key** (all events of order 42 in order).

By technology:

- **Kafka:** ordering guaranteed **within a partition**. Use the aggregate ID as the **partition key** so related events land in the same partition. Parallelism equals number of partitions.
- **Azure Service Bus:** use **sessions** (`SessionId` = aggregate ID) for FIFO per session with one consumer per session.
- **RabbitMQ:** order is preserved within a queue for a single consumer. Competing consumers, retries and redelivery break ordering. Consistent-hash exchange or single active consumer can help.

Defensive design when order can break:

- Include a **sequence number or aggregate version** in each event. The consumer ignores events older than the one it has applied, and parks or retries events that arrive too early (a gap).
- Make operations **commutative** or state-based where possible (last-write-wins with versions).
- Use **idempotent upserts** instead of strict inserts and updates.
- Avoid designs that need cross-aggregate ordering. Model that as a saga instead.

---

## Q38. Kafka vs RabbitMQ vs Azure Service Bus: how do you choose?

| | **Kafka** | **RabbitMQ** | **Azure Service Bus** |
|---|---|---|---|
| Model | Distributed **log** (append-only, retained) | Smart broker with exchanges and queues | Managed enterprise queues and topics |
| Retention | Time/size based; consumers can **replay** | Messages removed after ack (streams plugin offers log semantics) | Removed after completion (TTL, DLQ) |
| Throughput | Very high | Moderate to high | Moderate |
| Ordering | Per partition | Per queue (single consumer) | Per session |
| Routing | Topics and partitions (simple) | Rich routing (direct, topic, fanout, headers) | Topics with subscription filters |
| Consumers | Consumer groups track offsets | Competing consumers, push model | Competing consumers, peek-lock |
| Built-in features | Compaction, streams processing, connectors | Per-message TTL, priorities, DLX | **Sessions, DLQ, scheduled messages, duplicate detection, transactions** |
| Ops burden | High self-hosted, lower on managed offerings | Moderate | Lowest (fully managed) |
| Best for | Event streaming, event sourcing feeds, analytics, replay, high volume | Task queues, flexible routing, traditional messaging | Enterprise workflows on Azure, reliable commands and sagas |

Decision guidance:

- Need **replay, huge throughput, stream processing, long retention**: Kafka (or Azure Event Hubs).
- Need **rich routing and work queues** with moderate scale, on-premises: RabbitMQ.
- On **Azure** and want managed reliability with sessions, DLQ and scheduling: Service Bus.
- Many systems use **both**: a log for events and a queue-based broker for commands.

Also consider team skills, cloud strategy, cost and operational capacity. Avoid choosing Kafka just because it is popular.

---

## Q39. What is the difference between queues and topics, and what are competing consumers?

- **Queue (point-to-point):** each message is delivered to **one** consumer. Used for commands and work distribution.
- **Topic / pub-sub:** each message is delivered to **every subscription**. Each subscribing service gets its own copy (in its own queue), and consumers inside one subscription share the work. Used for events.

**Competing consumers.** Multiple instances read from the same queue or subscription. Each message is handled by only one instance. This gives horizontal scaling and resilience. Requirements:

- Handlers must be **stateless** and **idempotent**.
- Ordering is only guaranteed per key or session, if at all.
- In Kafka the equivalent is a **consumer group**, where each partition is consumed by at most one member, so useful parallelism is capped by partition count.

Typical layout: service Inventory has its own subscription on topic `order-events`, with N instances competing inside it. A different service, Analytics, has another subscription and receives all events independently.

---

## Q40. How do retries, dead-letter queues and poison messages work?

Failures come in two kinds:

- **Transient** (timeout, deadlock, downstream briefly unavailable): retrying usually succeeds.
- **Permanent** (invalid payload, bug, missing referenced data): retrying never helps.

Layered handling strategy:

1. **Immediate retries** (a few quick attempts) for very short glitches.
2. **Exponential backoff with jitter** (for example Polly) to avoid hammering a failing dependency and creating retry storms.
3. **Delayed redelivery / scheduled retry** (minutes, hours) for longer outages.
4. **Circuit breaker** to stop calling a dependency that is clearly down.
5. **Dead-letter queue (DLQ)** after a maximum delivery count. The message is moved aside with failure details, so it stops blocking the queue.
6. **Poison message** handling: a message that always fails must not block the rest. In ordered streams (sessions, Kafka partitions) a poison message can stall everything behind it, so route it to a retry topic or DLQ and move on.

Operational must-haves:

- **Alert** on DLQ depth and age.
- A **tooling or runbook to inspect, fix and replay (redrive)** dead-lettered messages after the cause is fixed.
- Log the **correlation ID and error** with each dead-letter.
- Distinguish error types in code: do not retry validation errors.

---

## Q41. How do you manage schema evolution and contract versioning for events?

Events are a **public API**. Consumers you do not control depend on them.

Rules:

1. **Backward compatibility first:** new producers must not break old consumers. Add optional fields. Never remove, rename or change the meaning or type of existing fields.
2. **Tolerant reader:** consumers ignore unknown fields and do not fail on additions.
3. **Explicit versioning** when a breaking change is unavoidable: a new event type or version (`OrderPlaced.v2`), published **alongside** the old one during a deprecation window, then retire the old one once all consumers migrate.
4. **Schema registry** (Confluent Schema Registry, Azure Schema Registry) with Avro/Protobuf/JSON Schema and compatibility modes (backward, forward, full) enforced in CI.
5. **Consumer-driven contract tests** (Pact) so producers know who breaks if they change.
6. **Metadata in headers:** event type, schema version, message ID, correlation ID, timestamp.
7. **Documentation and ownership:** a catalog (for example AsyncAPI) listing events, owners and consumers.
8. **Deprecation policy:** announce, dual-publish, measure remaining consumers, remove.

---

## Q42. Why not distributed transactions (2PC) across services?

**Two-phase commit** coordinates a commit across multiple resources: a prepare phase, then a commit phase.

Problems in microservice and cloud settings:

- **Blocking:** resources hold locks while waiting for the coordinator. A coordinator failure can leave them locked.
- **Availability and latency:** the transaction is only as available as all participants combined.
- **Tight coupling:** every participant must support the protocol (XA/MSDTC).
- **Unsupported:** many brokers, cloud services, NoSQL stores and SaaS APIs do not participate in 2PC.
- **Does not scale** across networks and regions.

The alternatives: **sagas** (local transactions plus compensation), **outbox/inbox** (reliable messaging with idempotency), and designing aggregate boundaries so most business operations are **single-service, single-transaction**. Accept **eventual consistency** between services, and make the business process tolerate it.

Honest note: for a small number of resources in one trusted environment, a distributed transaction might be acceptable. Say that, but explain why you would not build a platform on it.

---

## Q43. How do you get observability in an event-driven system?

With no single call stack, you must deliberately connect the dots.

1. **Correlation ID** that follows a business flow through every message (for example one per customer request or saga). Plus a **causation ID** (which message caused this one).
2. **Distributed tracing** with **OpenTelemetry**. Propagate W3C `traceparent` in message headers so traces continue across publish and consume, and spans cover producer, broker wait and consumer.
3. **Structured logging** including correlation ID, message ID, message type, aggregate ID and attempt number.
4. **Metrics:**
   - Consumer lag / queue depth, and age of the oldest message.
   - Processing time and throughput.
   - Error, retry and DLQ counts.
   - Outbox backlog and projection lag.
5. **Saga/process visibility:** a dashboard or query showing where each order or claim is, which step, and how long it has been there.
6. **Alerting** on stuck processes (no progress within a time budget), growing DLQ, and rising lag.
7. **Health checks** for broker connectivity and consumer liveness.
8. **Event catalog** documenting what produces and consumes each event.

---

## Q44. How do you scale consumers and handle backpressure?

**Scaling**

- Add consumer instances (competing consumers). On Kubernetes, use autoscaling on queue length or lag (for example KEDA).
- Parallelism is bounded: **Kafka** by partition count, **Service Bus sessions** by number of active sessions, **RabbitMQ** by queue and prefetch.
- Partition by a key with good distribution to avoid hot partitions (one very busy aggregate or tenant).

**Backpressure** (producers faster than consumers)

- Brokers buffer (that is a major benefit), but unbounded growth causes delay and cost.
- Limit consumer **concurrency and prefetch** so one instance does not grab more than it can process.
- **Rate-limit or throttle** calls to slow downstream systems (databases, partner APIs).
- Use **bulkheads** to isolate heavy message types from critical ones (separate queues).
- Set **TTLs** for stale messages that lose value.
- Apply **load shedding** or priority queues for overload.
- Monitor lag and scale before the backlog becomes unrecoverable.

Make sure scaling does not break correctness: more instances increase the chance of concurrent handling of related messages, so rely on keys, sessions, optimistic concurrency and idempotency.

---

## Q45. How do you test event-driven systems?

- **Unit tests:** handlers and aggregates with fake publishers (assert which events are emitted).
- **Consumer tests with an in-memory harness**, for example the MassTransit test harness to assert that a message was consumed or published.
- **Integration tests with real brokers and databases** via Testcontainers (RabbitMQ, Kafka, SQL Server).
- **Contract tests** (Pact or schema compatibility checks) between producers and consumers.
- **Idempotency tests:** deliver the same message twice and assert the result is unchanged.
- **Out-of-order tests:** deliver events in a different order than expected.
- **Failure injection:** broker down, handler throws, crash between steps. Verify retries, DLQ routing and outbox recovery.
- **Saga tests:** walk the state machine through success, failure and timeout paths, and verify compensations.
- **Eventually consistent assertions:** poll with a timeout instead of fixed sleeps.
- **End-to-end tests in a production-like environment** for critical flows, with trace-based assertions.
- **Replay tests:** reprocess a recorded stream and compare projections.

---

## Q46. What are the security considerations?

- **Authentication and authorization on the broker:** per-service credentials or managed identities, least privilege (a service may only publish or consume what it needs). Use ACLs, RBAC or SAS policies.
- **Encryption:** TLS in transit. Encryption at rest on the broker and event store.
- **Sensitive data in events:** minimize. Do not put secrets or unnecessary PII in payloads. If necessary, encrypt specific fields (and use crypto-shredding for erasure).
- **Message authenticity and integrity:** validate the sender's identity, and sign messages where trust boundaries require it.
- **Input validation:** consumers treat messages as untrusted input, and validate schema and values.
- **Tenant isolation:** carry tenant ID in messages, enforce it in consumers and projections, and separate topics or queues for sensitive tenants where needed.
- **Authorization context:** do not blindly trust the user identity in a message. Propagate claims in a verifiable way or re-authorize at the consumer.
- **Replay and duplicate attacks:** idempotency, timestamps, expiry.
- **Audit:** log who published and consumed what, and protect the logs.
- **Denial of service:** quotas and rate limits on producers, size limits on messages.

---

## Q47. Synchronous vs asynchronous communication: how do you choose?

| | Synchronous (HTTP/gRPC) | Asynchronous (messages/events) |
|---|---|---|
| Coupling | Temporal coupling (callee must be up now) | Decoupled in time |
| Latency | Immediate response | Result later |
| Consistency | Easier to reason about | Eventual |
| Failure | Propagates (cascading failures) | Buffered by broker |
| Debugging | Simpler call stack | Needs tracing |
| Use when | The caller **needs the answer now** (price lookup, auth check, validation against current data) | The work can happen later, many parties react, or you need resilience and scale |

Guidelines:

- **Queries** between services: usually synchronous, or better, avoid them with local data copies (event-carried state transfer).
- **Commands** that need an immediate yes/no: synchronous. Long-running or multi-step: asynchronous plus status tracking.
- **Notifications of facts:** asynchronous events.
- A chain of synchronous calls across many services (A calls B calls C calls D) multiplies latency and failure probability. This is a sign to rethink boundaries or go async.
- Mix them: a synchronous API accepts the request and returns `202`, then asynchronous processing completes the work and notifies the client.

Resilience for synchronous calls: timeouts, retries with backoff, circuit breakers, bulkheads (Polly or the .NET resilience libraries).

---

## Q48. What are the common event-driven anti-patterns?

1. **Events as remote procedure calls.** Publishing a "command in disguise" and waiting for a reply event, producing a slow, fragile RPC over messaging.
2. **Distributed monolith.** Services that must deploy together or share a database, connected by chatty events.
3. **Event spaghetti.** Choreography where no one can describe the end-to-end flow. No ownership or process view.
4. **Leaking internal models** as events (publishing domain or table-change events directly as the public contract).
5. **God events.** One huge event for everything, or events so generic (`EntityChanged`) that they carry no business meaning.
6. **Too granular events.** Dozens of tiny events per user action, flooding the broker.
7. **No idempotency or deduplication.**
8. **Dual writes without outbox.**
9. **Ignoring ordering assumptions.**
10. **No schema governance**, so breaking changes silently hit consumers.
11. **Missing observability** and no correlation IDs.
12. **No DLQ strategy**, so poison messages block queues or vanish.
13. **Treating the broker as a database.** Querying it for current state, or relying on indefinite retention without a design for it.
14. **Event sourcing everywhere** when not needed.
15. **Synchronous calls hidden inside handlers** that make an async design behave like a synchronous one.

---

# Part D: Design scenarios

## Q49. Design an e-commerce order flow using CQRS and event-driven design

**Requirements (assumed).** Customers place orders. The system must reserve stock, take payment, create a shipment and show order status. High read traffic on product and order views. Services: Orders, Inventory, Payments, Shipping, Notifications.

**Architecture**

1. **API gateway** receives `POST /orders` with an idempotency key.
2. **Orders service (command side):**
   - Validates input (FluentValidation), then the `Order` aggregate enforces invariants.
   - Saves the order with status `Pending` **and** an `OrderSubmitted` integration event into the **outbox** in one transaction.
   - Returns `201` (or `202`) with the order ID.
3. **Outbox relay** publishes `OrderSubmitted` to the broker.
4. **Saga orchestrator** (in Orders or a dedicated process manager) drives the flow:
   - Sends `ReserveInventory` to Inventory. On `InventoryReserved`, sends `ChargePayment`.
   - On `PaymentCompleted`, sends `CreateShipment`. On `ShipmentCreated`, marks the order `Confirmed`.
   - On `PaymentFailed`, sends `ReleaseInventory` (compensation) and marks the order `Cancelled`.
   - Timeout: if payment is not completed within, say, 15 minutes, cancel and compensate.
5. **Each participant** has an **inbox** for idempotent handling and its own outbox for the events it publishes.
6. **Query side:** a projection consumes order and shipment events and maintains an `OrderStatusView` (denormalized: customer name, lines, current status, tracking number) in a read store. `GET /orders/{id}` reads from it. Product catalog search uses a search index fed by catalog events.
7. **Notifications** subscribes to `OrderConfirmed` and `OrderCancelled` independently (pure choreography for this side effect).

**Data consistency decisions**

- Inventory must not oversell: enforce with an atomic conditional update (`UPDATE ... SET Available = Available - @q WHERE Available >= @q`) in the Inventory service, not by checking a read model.
- Order status in the UI is eventually consistent. The UI shows "Processing" and updates through SignalR or polling.

**Failure handling**

- Retries with backoff for transient errors, DLQ with alerts for poison messages.
- Compensation actions are idempotent (releasing the same reservation twice is safe).
- Saga state is persisted. Stuck sagas are visible on a dashboard.

**Observability:** correlation ID equals order ID, OpenTelemetry traces across services, metrics on saga duration and failure rates.

**Scaling:** Orders and Inventory scale independently, consumers compete on queues, Kafka or Service Bus partitions/sessions are keyed by order ID for ordering per order.

**Trade-offs to state:** complexity of a saga versus a monolithic transaction, eventual consistency for status, operational cost of the broker. Mention that if all this lived in one service and one database, a single transaction would be simpler and preferable.

---

## Q50. Design an insurance claim processing system

**Requirements (assumed).** Customers submit claims with documents. Claims pass through validation, fraud check, assessment (possibly manual), approval and payout. Full audit trail is required. Auditors need to know who changed what and why. Adjusters need fast, filterable work queues.

**Why this domain suits the patterns**

- **Audit and history are core business requirements**, so event sourcing is a reasonable fit for the `Claim` aggregate (`ClaimSubmitted`, `DocumentsAttached`, `FraudCheckPassed`, `AssessmentCompleted`, `ClaimApproved`, `PayoutIssued`, `ClaimRejected` with reason).
- **Complex read needs** (adjuster queues by status, region, amount, age; dashboards; search) are served by **CQRS projections**.

**Design**

1. **Command side:** `SubmitClaim`, `AttachDocument`, `AssignAdjuster`, `ApproveClaim`, `RejectClaim`. The `Claim` aggregate enforces rules (cannot approve above the adjuster's limit, cannot pay before approval). Events are appended with expected-version concurrency.
2. **Process management:** a saga or workflow (orchestration, since this is long-running with human steps and timeouts) coordinates: validate policy (Policy service), run the fraud check (async, third party), wait for adjuster assessment (human task with SLA timer), approve, then issue payout (Payments service).
3. **Integration events:** `ClaimSubmitted`, `ClaimApproved`, `PayoutRequested` are mapped from domain events as stable contracts. Published through an outbox.
4. **Projections:** `AdjusterWorkQueue` (SQL, indexed for filtering), `ClaimTimelineView` (history for customer and auditors, straight from events), `ClaimsDashboard` (aggregates), and a search index.
5. **External systems:** the fraud check and payment gateway are unreliable and slow. Use async requests with timeouts, retries, circuit breakers and idempotency keys.
6. **Documents:** store files in blob storage and put only references in events.
7. **Privacy:** minimize personal data in events, use crypto-shredding for personal fields, and enforce role-based access in projections and APIs.
8. **Compensation:** if a payout fails after approval, the saga retries (payout is a retriable step after the pivot) and escalates to a human queue after a limit. If a claim is wrongly approved, a compensating event (`ApprovalRevoked`) is appended, never an edit.

**If the team prefers to avoid event sourcing:** use a state-based aggregate with an **append-only `ClaimHistory` table** (or SQL Server temporal tables) plus the outbox for integration events. This gives most of the audit value at lower complexity.

**Non-functional:** SLA timers and alerts for stuck claims, DLQ monitoring, correlation ID equals claim ID, replayable projections for new reports.

---

## Q51. What clarifying questions should you ask before designing?

Asking these shows senior judgment and prevents over-engineering:

**Business and consistency**
- Which operations need immediate, strong consistency, and which can be eventually consistent? How stale can a read be?
- What is the cost of a duplicate or a lost operation (money, legal)?
- Is a full audit trail or history a business or regulatory requirement?

**Scale and performance**
- Expected read-to-write ratio, peak load, data volume, growth?
- Latency targets for key operations?

**Domain**
- Are there complex business rules and invariants, or mostly CRUD?
- Which operations span multiple aggregates or services?
- Are there long-running or human-in-the-loop processes?

**Team and operations**
- Team size and experience with messaging, CQRS and event sourcing?
- Existing infrastructure (cloud, broker, databases)? Budget and operational capacity?
- Monitoring and on-call maturity?

**Evolution and compliance**
- How often will contracts change? Who are the consumers?
- Data retention, privacy and deletion requirements?
- Multi-tenancy and data residency?

State your **assumptions out loud**, then pick the **simplest design** that meets them, and explain what would make you move to something more complex.

---

# Part E: Cheat sheet

## Pattern quick reference

| Pattern | One-sentence purpose | Key risk |
|---|---|---|
| **CQRS** | Separate write and read models so each is optimized | Complexity, eventual consistency |
| **Event sourcing** | Store changes as immutable events, derive state by replay | Versioning, querying, GDPR |
| **Domain event** | Signal inside a bounded context that something happened | Over-reliance on in-process dispatch |
| **Integration event** | Stable, versioned contract between services | Breaking consumers |
| **Outbox** | Atomically save state and the message to publish | Relay duplicates and backlog |
| **Inbox / idempotent consumer** | Process each message effectively once | Table growth, race conditions |
| **Saga** | Multi-service business transaction through local steps and compensations | Lack of isolation, complexity |
| **Choreography** | Services react to each other's events | Invisible flow |
| **Orchestration** | Central coordinator drives the flow | Orchestrator becomes a god object |
| **DLQ** | Park messages that cannot be processed | Unmonitored dead letters |
| **Snapshot** | Speed up aggregate loading in event sourcing | Premature optimization |
| **Projection** | Build read models from events | Lag, rebuild cost |

## Consistency decision guide

- Needs **immediate correctness** (stock decrement, balance check)? Check against the **write model or database**, with atomic operations or constraints.
- Display data that can lag? Use the **read model**.
- Across services? **Saga + events**, with compensation.

## Delivery and ordering reminders

- Assume **at-least-once**. Make everything **idempotent**.
- Order only **per key**. Add **versions** to detect gaps and stale events.
- **Outbox** on publish, **inbox** on consume.

## Interview closing lines

- "I would start with the simplest separation that solves the actual problem and add eventual consistency only where the business can accept it."
- "Every message can arrive twice or late, so handlers are idempotent and tolerant of ordering issues."
- "I would invest in observability and a rebuild/replay path from day one, since that is what makes event-driven systems operable."

## Glossary

- **Aggregate:** a cluster of domain objects treated as one consistency unit, accessed through its root.
- **Bounded context:** an explicit boundary within which a domain model and its language are consistent.
- **Eventual consistency:** different parts of the system converge to the same state over time, not instantly.
- **Idempotency:** performing an operation multiple times has the same effect as once.
- **Projection:** a read model derived from events.
- **Compensating action:** a business-level undo for a completed step.
- **Poison message:** a message that repeatedly fails processing.
- **Consumer lag:** how far behind a consumer is from the latest message.
- **CDC (Change Data Capture):** streaming database changes from the transaction log.
