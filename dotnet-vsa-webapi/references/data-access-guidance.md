# Data access guidance

This file defines how to choose and use EF Core and Dapper in a Vertical Slice architecture.

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
