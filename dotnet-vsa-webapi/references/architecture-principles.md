# Architecture principles

This file defines the non-negotiable design rules for building .NET web applications with Vertical Slice Architecture in a pragmatic, production-ready style.

## Primary law

**Each HTTP request is an independent slice.**

A slice is usually one of:

- Command: changes state
- Query: reads state

A slice should contain only the code needed for that use case.

That usually means colocating:

- endpoint
- request DTO / command / query
- validator
- handler / use-case orchestration
- response DTO
- local mapping
- local query object / SQL
- tests for that slice

## Core structure

Prefer this shape for a single application:

```text
src/
  Web/
    Program.cs
    Features/
      Shipments/
        CreateShipment/
          Endpoint.cs
          Request.cs
          Validator.cs
          Handler.cs
          Response.cs
        GetShipmentById/
          Endpoint.cs
          Query.cs
          Handler.cs
          Response.cs
        Shared/
          ShipmentMappings.cs
      Billing/
        ...
    Domain/
      Shipments/
        Shipment.cs
        ShipmentItem.cs
        ShipmentErrors.cs
      Billing/
        Invoice.cs
        ...
    Infrastructure/
      Persistence/
      Auth/
      Messaging/
      Observability/
```

For a modular monolith, prefer:

```text
src/
  Modules/
    Shipments/
      Domain/
      Features/
      Infrastructure/
      PublicApi/
    Billing/
      Domain/
      Features/
      Infrastructure/
      PublicApi/
  Web/
    Program.cs
```

## What “feature-first” means

Bad primary organization:

```text
Controllers/
Services/
Repositories/
Dtos/
Validators/
```

Good primary organization:

```text
Features/
  Orders/
    CreateOrder/
    CancelOrder/
    GetOrderById/
```

The first shape optimizes for technical categories.
The second shape optimizes for business change.

## Slice anatomy

A good slice usually follows this path:

1. HTTP endpoint receives request.
2. Request is validated close to the endpoint.
3. Handler executes use-case logic.
4. Domain enforces invariants where it adds clarity.
5. Infrastructure dependencies are invoked through direct adapters or meaningful abstractions.
6. Handler returns `Result` or `Result<T>`.
7. Endpoint maps result to explicit HTTP response.

## Thin endpoint rule

An endpoint should not contain business workflow logic.

An endpoint may:

- bind route/body/query parameters
- resolve validator/handler from DI
- call validation
- call handler
- map result to HTTP response
- attach metadata (`WithName`, `Produces`, auth policy, tags)

An endpoint should not:

- implement transactional workflow
- contain database logic
- call multiple unrelated services directly
- hide business decisions in inline lambdas
- throw expected business exceptions

## Clean Architecture inside a slice

Inside a slice or module, preserve inward dependency direction.

### Domain

Put here:

- entities
- value objects
- domain services when truly needed
- domain errors
- invariants
- business terminology

Avoid:

- web concerns
- ORM-only behavior leaking everywhere
- DTO semantics

### Use-case / Application logic

Put here:

- handler orchestration
- transaction boundaries
- application decisions
- coordination with infrastructure
- mapping from domain to response

Avoid:

- giant shared service classes
- business rules spread across endpoints and infrastructure

### Infrastructure edge

Put here:

- EF Core DbContext
- Dapper connection factory
- external API clients
- auth adapters
- file/storage adapters
- message bus adapters

Avoid:

- leaking infrastructure types through the whole app
- hiding everything behind “one repository to rule them all”

## Interfaces: when they are meaningful

Good reasons to introduce interfaces:

- external gateway boundaries
- inter-module APIs
- time and user context
- connection factories
- repositories around aggregate persistence when multiple implementations or stable seams matter
- policies/strategies with genuine variation

Bad reasons:

- interface-per-class habit
- “future proofing”
- hiding a single EF Core query with no alternative
- adding a repository only because “Clean Architecture says so”

### Practical rule

Create an interface only when at least one of these is true:

- it isolates an unstable dependency
- it narrows the surface exposed to other code
- it enables local replacement in tests without mocking the universe
- it represents a real business or module contract
- it prevents cross-slice leakage

## Shared code policy

Vertical slices should not devolve into copy-paste architecture, but they also must not centralize everything too early.

### Allowed extraction levels

#### 1. Slice-local

Keep logic inside the slice if it changes only with that request.

#### 2. Feature-shared

If two or three closely related slices in the same feature reuse logic for the **same reason**, extract into:

```text
Features/Shipments/Shared/
```

Examples:

- projection helpers for shipment DTOs
- shared validator fragments for shipment items
- common domain-specific parsing

#### 3. Module-wide infrastructure

If multiple slices in one module share a persistence/auth/messaging concern, place it in the module infrastructure.

Examples:

- `IDbConnectionFactory`
- `ShipmentsDbContext`
- `IShipmentModuleApi`

#### 4. App-wide cross-cutting

Reserve app-wide shared code for truly cross-cutting concerns:

- result/error primitives
- ProblemDetails mapping
- auth policy registration
- common middleware
- observability bootstrapping

### Extraction rule

Extract only when:
- duplication is real
- the code has the same reason to change
- the extraction lowers cognitive load

Do not extract because two code blocks merely “look similar”.

## C# style defaults

### Do not add `sealed` by default

Do not mark classes or records as `sealed` unless there is a specific reason (e.g., security-sensitive class, performance-critical hot path where devirtualization matters).
The default is simply `public class` / `public record`.

### Prefer primary constructors

Use primary constructors for dependency injection, handlers, services, and simple data classes.
They reduce boilerplate and keep the class declaration concise.

Good:
```csharp
public class CreateShipmentHandler(
    AppDbContext dbContext,
    IShipmentNumberGenerator numberGenerator,
    TimeProvider timeProvider,
    ILogger<CreateShipmentHandler> logger)
    : ICreateShipmentHandler
{
    // use dbContext, numberGenerator, etc. directly
}
```

Avoid:
```csharp
public class CreateShipmentHandler : ICreateShipmentHandler
{
    private readonly AppDbContext _dbContext;
    private readonly IShipmentNumberGenerator _numberGenerator;

    public CreateShipmentHandler(
        AppDbContext dbContext,
        IShipmentNumberGenerator numberGenerator)
    {
        _dbContext = dbContext;
        _numberGenerator = numberGenerator;
    }
}
```

Primary constructors work for:
- handlers and services (DI injection)
- DbContext subclasses: `public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)`
- infrastructure adapters: `public class NpgsqlConnectionFactory(IOptions<PostgresOptions> options) : IDbConnectionFactory`
- simple value types and DTOs when records don't fit

Use traditional constructors only when:
- the constructor has complex initialization logic
- you need to validate/transform parameters before assignment
- the class is a domain entity with factory methods and a private constructor

## Naming rules

Names must scream the domain and use case.

Prefer:

- `CreateShipment`
- `CancelInvoice`
- `ReserveInventory`
- `ShipmentAlreadyExists`
- `GetInvoiceByNumber`

Avoid:

- `Handler`
- `Service`
- `Manager`
- `Process`
- `Execute`

Use generic suffixes only when the file’s surrounding path already gives context.

## Request-specific handlers over god-services

Prefer this:

```text
Features/Orders/CreateOrder/CreateOrderHandler
Features/Orders/CancelOrder/CancelOrderHandler
Features/Orders/GetOrderById/GetOrderByIdHandler
```

Over this:

```text
Services/OrderService.cs
```

with 40 public methods.

## Dependency direction rules

Allowed:

- Endpoint -> Handler
- Handler -> Domain + infrastructure seams
- Infrastructure -> framework/db/network libraries
- One module -> another module’s `PublicApi`

Avoid:

- Domain -> Infrastructure
- One feature directly touching another feature’s internals
- Module A querying Module B’s database tables
- giant “Shared” helpers used by everything

## Command and query separation

You do not need a messaging library to separate commands and queries.

It is enough that:
- write use cases are modeled as commands/operations
- read use cases are modeled as queries/projections
- they can evolve independently
- they may use different data access tools

This skill encourages:
- EF Core for commands and transactional aggregate work
- Dapper for read-optimized projections where justified

## Patterns that fit this architecture

Use only when justified.

### Good fits

- Strategy  
  For varying pricing, tax, policy, or provider behavior.

- Factory  
  For constructing aggregates/value objects that require non-trivial creation logic.

- Specification  
  For reusable query predicates or business predicates when duplication is real.

- Policy  
  For named business decisions, especially authorization or workflow guards.

- Decorator  
  For explicit cross-cutting around stable seams, not magic pipeline worship.

- Facade  
  For inter-module public APIs.

### Use cautiously

- Base classes
- pipeline frameworks
- generic base handlers
- event buses inside a monolith for everything

## Refactor strategy: layered app -> vertical slices

Do not rewrite the whole app at once.

### Migration playbook

1. Pick one endpoint/use case.
2. Create a feature folder for it.
3. Move request/validator/handler/response close together.
4. Leave existing shared services in place temporarily.
5. Inline only the parts needed for that slice.
6. Introduce Result mapping.
7. Repeat for adjacent slices.
8. Extract only the shared logic that proves stable.
9. Collapse old layer folders once empty.

### Good intermediate state

A mixed codebase is acceptable during migration if:
- new work lands in slices
- dependencies become clearer over time
- behavior is preserved
- the architecture trend is improving

## Review checklist

When reviewing a repository, ask:

- Can I find everything for one request in one place?
- Does the endpoint stay thin?
- Are status codes explicit?
- Are expected errors result-based?
- Is validation close to the slice?
- Are interfaces meaningful?
- Is there cross-slice leakage?
- Is shared code earned or speculative?
- Are commands and queries free to evolve differently?
- Would a business change stay local?

## Default answer to “where should this code go?”

- If it belongs to one request: inside that slice.
- If it belongs to closely related requests: feature shared.
- If it is infrastructure for a module: module infrastructure.
- If it is cross-cutting for the whole app: app root/shared cross-cutting.
- If unsure: keep it local first.
