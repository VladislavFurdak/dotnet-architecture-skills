# Data access guidance

This file defines how to choose and use EF Core and Dapper in a Vertical Slice architecture.

## NEVER use InMemoryDatabase

**`UseInMemoryDatabase` is explicitly banned for new projects.** It does not support transactions, referential integrity, migrations, or SQL features. It creates false confidence when tests pass against InMemory but fail against a real database.

Always use a real PostgreSQL database via Aspire:
```csharp
// Correct — real database via Aspire
builder.AddNpgsqlDbContext<AppDbContext>("resource-name");

// BANNED — never do this
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseInMemoryDatabase("...")); // ← NEVER
```

## Default position

Both EF Core and Dapper are valid.
Choose per slice, not by ideology.

A mixed strategy is often best:
- EF Core for command/write-side aggregate work
- Dapper for read-side projections and SQL-shaped queries

## Decision matrix

## Choose EF Core when

- you are modifying aggregates
- you need change tracking
- you need optimistic concurrency support
- the write workflow spans related entities
- the domain model benefits from navigations/value objects
- migrations are part of your workflow
- the query is not performance-critical or SQL-shaped enough to justify hand-written SQL

Typical EF Core slices:
- CreateShipment
- CancelOrder
- ApproveInvoice
- ReserveInventory

## Choose Dapper when

- you are reading projection DTOs
- the query is join-heavy or reporting-style
- the result shape does not align well with aggregates
- you want tight SQL control
- the endpoint is read-hot and latency sensitive
- you need database-specific SQL features
- the read model is intentionally separate from the write model

Typical Dapper slices:
- GetShipmentById
- SearchInvoices
- DashboardSummary
- ExportOrders

## Mixed within the same module

This is acceptable and often ideal.

Example:
- `CreateShipment` uses EF Core
- `GetShipmentById` uses Dapper

They still belong to the same feature/module.

## EF Core rules

## Prefer direct DbContext access in slices

For EF Core, do not force a generic repository by default.

Good:
```csharp
public class CreateShipmentHandler(ShipmentsDbContext dbContext, ...)
```

Questionable:
```csharp
public class CreateShipmentHandler(IRepository<Shipment> repository, ...)
```

unless the repository represents a real aggregate boundary and adds value.

## When a repository is justified with EF Core

A repository can make sense when:
- it represents an aggregate boundary
- it hides a stable, domain-meaningful persistence contract
- multiple handlers need the same aggregate persistence semantics
- it protects cross-module callers from persistence details

Example:
```csharp
public interface IShipmentRepository
{
    Task<bool> ExistsForOrderAsync(string orderId, CancellationToken ct);
    Task AddAsync(Shipment shipment, CancellationToken ct);
}
```

Still avoid:
- `IRepository<T>`
- `GetAll()`
- `Find(Expression<Func<T, bool>> predicate)` everywhere
- repositories that merely mirror `DbSet<T>`

## EF Core slice practices

- keep query logic close to the slice unless truly shared
- project to DTOs at query time when the endpoint needs DTOs
- avoid loading large graphs unless needed
- keep transactions explicit when workflows require them
- do not expose tracked entities as HTTP contracts
- favor `AsNoTracking()` for read-only EF Core queries

## EF Core performance rules

Follow these rules for every EF Core query. Performance is not optional — it is a baseline expectation.

### 1. Fix N+1 queries

Use `.Include()` for known related data. Never load related entities in a loop.

Good:
```csharp
var orders = await dbContext.Orders
    .Include(o => o.Customer)
    .ToListAsync(cancellationToken);
```

Bad:
```csharp
var orders = await dbContext.Orders.ToListAsync(cancellationToken);
foreach (var order in orders)
{
    order.Customer = await dbContext.Customers.FindAsync(order.CustomerId); // N+1
}
```

### 2. Project to DTOs — do not fetch full entities for reads

```csharp
var result = await dbContext.Orders
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummaryResponse(o.Id, o.Number, o.Total, o.CreatedAt))
    .ToListAsync(cancellationToken);
```

Projection eliminates change tracking overhead and fetches only needed columns.

### 3. Use `AsNoTracking()` for read-only queries

```csharp
var product = await dbContext.Products
    .AsNoTracking()
    .FirstOrDefaultAsync(p => p.Id == productId, cancellationToken);
```

If the entity will not be modified, disable tracking. This saves memory and CPU.

### 4. Use `AsSplitQuery()` for multiple includes

When a query has multiple `.Include()` calls, EF Core generates a single SQL with JOINs that can cause a cartesian explosion. Split it:

```csharp
var order = await dbContext.Orders
    .AsSplitQuery()
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .FirstOrDefaultAsync(o => o.Id == orderId, cancellationToken);
```

Use split queries when including two or more collection navigations.

### 5. Use batch operations for bulk updates/deletes (EF Core 7+)

Do not load entities just to delete or update them:

```csharp
// Good: single SQL DELETE
await dbContext.Products
    .Where(p => p.IsDiscontinued)
    .ExecuteDeleteAsync(cancellationToken);

// Good: single SQL UPDATE
await dbContext.Products
    .Where(p => p.CategoryId == oldCategoryId)
    .ExecuteUpdateAsync(s => s.SetProperty(p => p.CategoryId, newCategoryId), cancellationToken);
```

### 6. Paginate in the database

```csharp
var page = await dbContext.Orders
    .OrderBy(o => o.CreatedAt)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync(cancellationToken);
```

Never call `.ToList()` first and then paginate in memory.

### 7. Use compiled queries for hot paths

For queries executed very frequently with the same shape:

```csharp
private static readonly Func<AppDbContext, Guid, Task<Product?>> GetById =
    EF.CompileAsyncQuery((AppDbContext ctx, Guid id) =>
        ctx.Products.FirstOrDefault(p => p.Id == id));
```

### 8. Configure indexes in entity configuration

```csharp
modelBuilder.Entity<Order>(entity =>
{
    entity.HasIndex(o => o.CustomerId);
    entity.HasIndex(o => new { o.Status, o.CreatedAt });
});
```

Index columns used in `Where`, `OrderBy`, and `Join` predicates.

### 9. Use `FindAsync` for single-entity lookup by primary key

```csharp
var product = await dbContext.Products.FindAsync([productId], cancellationToken);
```

`FindAsync` checks the local cache first, avoiding a round-trip if the entity is already tracked.

### 10. Avoid lazy loading

Do not configure lazy loading proxies. Use explicit `.Include()` or projection instead.

## MANDATORY: Predicate composition — do not embed external variable checks in Where clauses

When building dynamic filters, **do not cram optional conditions into a single `.Where()` expression with external variable checks**. This produces unreadable LINQ, generates suboptimal SQL (the database evaluates conditions that are always false), and is hard to maintain.

### Bad: monolithic predicate with external variable checks

```csharp
contactsQuery = contactsQuery.Where(c =>
    (!string.IsNullOrWhiteSpace(alias) && c.Alias.Contains(alias!)) ||
    (!string.IsNullOrWhiteSpace(name) && c.Name.Contains(name!)) ||
    (hasChat.HasValue && hasChat.Value == (c.ChatId != null)) ||
    (!string.IsNullOrWhiteSpace(lastMessageContains) &&
        c.LastMessage != null && c.LastMessage.Contains(lastMessageContains!)));
```

Problems:
- external variables (`alias`, `name`, etc.) leak into the expression tree
- EF Core may not optimize away branches that are always false
- generated SQL is bloated and hard to debug
- adding a new filter requires editing a fragile compound expression

### Good: compose the predicate incrementally

For **AND** semantics (all conditions must match), chain `.Where()` calls:

```csharp
if (!string.IsNullOrWhiteSpace(alias))
{
    query = query.Where(c => c.Alias != null && c.Alias.Contains(alias));
}

if (!string.IsNullOrWhiteSpace(name))
{
    query = query.Where(c => c.Name != null && c.Name.Contains(name));
}

if (hasChat.HasValue)
{
    var hasChatValue = hasChat.Value;
    query = query.Where(c => (c.ChatId != null) == hasChatValue);
}

if (!string.IsNullOrWhiteSpace(lastMessageContains))
{
    query = query.Where(c =>
        c.LastMessage != null && c.LastMessage.Contains(lastMessageContains));
}
```

Each `.Where()` adds a clean SQL `AND` clause. Unused filters produce no SQL at all.

### Good: compose with Union for OR semantics

When filters represent alternative match criteria (any condition should match), build separate queries and combine with `.Union()`:

```csharp
var source = contactsQuery;
IQueryable<Contact>? result = null;

if (!string.IsNullOrWhiteSpace(alias))
{
    var q = source.Where(c => c.Alias != null && c.Alias.Contains(alias));
    result = result == null ? q : result.Union(q);
}

if (!string.IsNullOrWhiteSpace(name))
{
    var q = source.Where(c => c.Name != null && c.Name.Contains(name));
    result = result == null ? q : result.Union(q);
}

if (hasChat.HasValue)
{
    var hasChatValue = hasChat.Value;
    var q = source.Where(c => (c.ChatId != null) == hasChatValue);
    result = result == null ? q : result.Union(q);
}

if (!string.IsNullOrWhiteSpace(lastMessageContains))
{
    var q = source.Where(c =>
        c.LastMessage != null && c.LastMessage.Contains(lastMessageContains));
    result = result == null ? q : result.Union(q);
}

contactsQuery = result ?? source;
```

### Rules for predicate composition

- **Evaluate filter presence outside the expression tree** — check `string.IsNullOrWhiteSpace()`, `.HasValue`, etc. before building the `.Where()`
- **Chain `.Where()` for AND** — each call narrows the result, produces clean SQL
- **Use `.Union()` for OR** — each branch is a separate SQL query combined with `UNION`
- **Capture variables before the lambda** — e.g., `var hasChatValue = hasChat.Value;` then use `hasChatValue` in the expression to avoid closure over nullable
- **Never mix external control flow (`&&`, `||` on C# variables) with entity predicates in a single expression**

## Dapper rules

## Keep SQL local to the slice or module

Dapper exists to make explicit SQL easy.
Do not hide it behind layers that destroy that benefit.

Good:
```text
Features/Shipments/GetShipmentById/Handler.cs
Features/Shipments/SearchShipments/Sql.cs
```

Acceptable:
```text
Modules/Shipments/Infrastructure/Queries/
```

Bad:
```text
Shared/Sql/
Common/Queries/
```

unless it is a truly shared, stable module concern.

## Use a connection factory interface

A narrow abstraction is useful here.

Example:
```csharp
public interface IDbConnectionFactory
{
    Task<IDbConnection> OpenConnectionAsync(CancellationToken cancellationToken);
}
```

Why:
- stable seam
- simple testing replacement
- keeps provider wiring outside the slice

## Dapper query practices

- select only needed columns
- map straight to response DTOs
- keep SQL readable and named
- use parameters, never string interpolation
- keep transaction handling explicit when needed
- do not return data access rows all the way to the API

## Query objects vs inline SQL

Use inline SQL in the handler when:
- the query is short
- the slice owns it fully

Extract to a nearby `Sql.cs` or query object when:
- the SQL is long
- the SQL is reused by related read slices
- a named query improves readability

## Duplication and extraction rules

Extract shared data-access code only when:
- multiple slices use the exact same predicate/projection
- it changes for the same reason
- the extraction improves navigation

For EF Core, shared pieces can be:
- query extensions
- specifications
- projection expressions
- a small repository around a real aggregate seam

For Dapper, shared pieces can be:
- a named SQL file/string
- a query object
- a module-level query helper

## Persistence boundaries in modular monoliths

For modular monoliths:
- each module owns its schema/tables
- each module owns its DbContext
- modules do not query each other’s tables directly
- cross-module interaction happens through public APIs or events

## Transactions

## Inside a slice

Keep the transaction where the use case is orchestrated.

Examples:
- EF Core `SaveChangesAsync` for standard aggregate writes
- explicit transaction for multi-step command logic
- Dapper transaction when a query/update batch needs atomicity

## Across modules

Do not reach for distributed-transaction fantasies inside a modular monolith.

Prefer:
- local transaction + event/outbox when necessary
- module API boundaries
- eventual consistency where appropriate

## Performance heuristics

Choose Dapper earlier when:
- the endpoint is read-hot
- the projection is flat and SQL-shaped
- EF Core would load too much or create awkward mapping

Stick with EF Core when:
- the operation is aggregate-rich
- the write model matters more than raw query speed
- change tracking and migrations save significant effort

## Anti-patterns

- generic repository over everything
- returning entities directly from read endpoints
- cross-module table access
- dumping all SQL into a global infrastructure folder
- pretending every read must use the same persistence style as writes
- hiding SQL behind meaningless “query service” abstractions
- mapping database rows to entities to DTOs for a simple read projection

## Review checklist

- Why was EF Core or Dapper chosen for this slice?
- Is the choice aligned with write vs read shape?
- Is SQL local and readable?
- Is DbContext usage direct and clear?
- Are repository abstractions meaningful?
- Are responses projected to DTOs instead of leaking entities?
- Are module boundaries preserved?
