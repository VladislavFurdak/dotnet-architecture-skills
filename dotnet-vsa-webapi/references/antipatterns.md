# Anti-patterns

Use this file when reviewing a repository or planning a refactor.

## Architecture anti-patterns

## 1. Layer-first primary organization

Symptoms:
- `Controllers/`, `Services/`, `Repositories/`, `Dtos/`
- one feature scattered across 6 folders

Why it hurts:
- poor discoverability
- business change spans too many files
- onboarding follows mechanics, not use cases

Refactor target:
- move one use case at a time into `Features/<Entity>/<UseCase>/`

## 2. Fat endpoints or controllers

Symptoms:
- validation, workflow, database access, and mapping all inside endpoint body

Why it hurts:
- hard to test
- hard to reuse
- HTTP and business concerns are tangled

Refactor target:
- endpoint validates and delegates
- handler owns workflow

## 3. God-service

Symptoms:
- `OrderService` with dozens of methods
- unrelated workflows coupled together

Why it hurts:
- change blast radius
- poor cohesion
- merge conflicts

Refactor target:
- request-specific handlers per use case

## 4. Generic repository everywhere

Symptoms:
- `IRepository<T>`
- `GetAll`, `Find`, `Update`, `Delete`
- EF Core hidden behind weak abstractions

Why it hurts:
- erases use-case intent
- adds ceremony
- often leaks IQueryable or useless indirection

Refactor target:
- direct DbContext in slices
- repository only for real aggregate seams

## 5. Shared dumping ground

Symptoms:
- `Shared/`, `Common/`, `Utils/` filled with unrelated code
- no ownership boundaries

Why it hurts:
- hidden coupling
- discoverability collapse
- accidental cross-feature dependency

Refactor target:
- keep logic local
- extract only to feature-shared, module infrastructure, or true cross-cutting folders

## 6. Cross-slice leakage

Symptoms:
- one feature directly references another feature’s internals
- handlers call each other freely

Why it hurts:
- fragile coupling
- hidden dependencies
- breaks local change

Refactor target:
- shared logic extraction or module public API
- never feature-to-feature internal reach-through

## 7. Cross-module database access

Symptoms:
- module A queries module B tables directly

Why it hurts:
- destroys modular boundary
- blocks future extraction
- creates invisible distributed ownership

Refactor target:
- module public API or events

## 8. MediatR-shaped indirection without value

Symptoms:
- command/handler indirection adds navigation cost
- handlers call handlers
- debugging requires tracing pipeline magic

Why it hurts:
- unnecessary hops
- poor local reasoning

Refactor target:
- direct handler/service invocation
- explicit decorators only where justified

## 9. AutoMapper-first architecture

Symptoms:
- hidden mapping rules
- DTO/entity conversions spread through profiles
- hard-to-see projection cost

Why it hurts:
- mapping logic becomes implicit
- runtime surprises
- difficult debugging

Refactor target:
- explicit local mapping
- projection to DTO at query time

## 10. Exception-driven business flow

Symptoms:
- domain/business conditions thrown as exceptions
- try/catch used for normal outcomes

Why it hurts:
- noisy control flow
- performance overhead
- unstable error contracts

Refactor target:
- `Result` / `Error` for expected outcomes

## HTTP/API anti-patterns

## 11. Inconsistent status code mapping

Symptoms:
- same business failure returns 400 in one place, 409 in another

Why it hurts:
- client confusion
- poor contract stability

Refactor target:
- shared error taxonomy + mapping policy

## 12. 200 OK for everything

Symptoms:
- failures encoded only in body fields like `success: false`

Why it hurts:
- HTTP contract loses meaning
- tooling/OpenAPI less useful

Refactor target:
- explicit status codes
- ProblemDetails-compatible failures

## 13. Returning entities directly

Symptoms:
- EF entities returned from API

Why it hurts:
- contract leakage
- accidental data exposure
- serialization surprises

Refactor target:
- response DTOs / read projections

## 14. Validation too far from the slice

Symptoms:
- giant central validation layer
- magic auto-validation nobody can trace

Why it hurts:
- poor locality
- harder debugging

Refactor target:
- validator beside request
- explicit invocation

## Data access anti-patterns

## 15. Dapper hidden behind pseudo-ORM abstractions

Symptoms:
- SQL wrapped in generic services with no locality

Why it hurts:
- lose Dapper’s clarity advantage
- harder query ownership

Refactor target:
- keep SQL near the slice/module

## 16. EF Core used as a read-model dump truck

Symptoms:
- huge tracked graphs for simple read DTOs
- awkward includes and post-mapping

Why it hurts:
- performance
- complexity

Refactor target:
- project directly or use Dapper for projection-heavy reads

## 17. Query duplication extracted too early

Symptoms:
- giant reusable query builders
- premature abstraction

Why it hurts:
- more indirection than value

Refactor target:
- duplicate locally until same-reason reuse is proven

## Interfaces and dependency anti-patterns

## 18. Interface-per-class cargo cult

Symptoms:
- every class has an interface regardless of seam value

Why it hurts:
- noise
- fake abstraction
- lower discoverability

Refactor target:
- keep interfaces only for real boundaries

## 19. Service locator

Symptoms:
- classes resolve services from provider at runtime
- hidden dependencies

Why it hurts:
- untestable
- opaque behavior

Refactor target:
- constructor injection / explicit parameters

## 20. Static mutable state

Symptoms:
- process-wide mutable caches/config/state in static classes

Why it hurts:
- hidden coupling
- test pollution
- concurrency bugs

Refactor target:
- scoped/singleton services with explicit contracts

## Observability and ops anti-patterns

## 21. Duplicate logging at every layer

Symptoms:
- endpoint logs request
- handler logs same request
- repository logs same failure again

Why it hurts:
- noisy logs
- expensive and unreadable telemetry

Refactor target:
- one request log
- meaningful event logs only

## 22. Liveness and readiness conflated

Symptoms:
- one `/health` endpoint used for everything
- dependency failure restarts healthy process

Why it hurts:
- false restarts
- unstable deployments

Refactor target:
- separate live/ready endpoints

## 23. Secrets/config strings scattered in code

Symptoms:
- direct config lookups everywhere
- magic section names everywhere

Why it hurts:
- drift
- testing pain
- brittle configuration

Refactor target:
- strongly typed options

## Minimal API binding anti-patterns

## 26. Missing `[FromBody]` on DELETE endpoints

Symptoms:
- DELETE endpoint accepts a JSON body but parameter lacks `[FromBody]`
- runtime binding failure or null parameter on startup

Why it hurts:
- Minimal API infers body only for POST, PUT, PATCH — not DELETE
- causes startup errors or silent null binding at runtime

Refactor target:
- add `[FromBody]` to any DELETE handler parameter that expects a JSON body

## 27. Blindly propagating CancellationToken through multi-write operations

Symptoms:
- `CancellationToken` passed to every async call including multiple sequential writes without a transaction
- client disconnect mid-flow leaves partial commits

Why it hurts:
- data inconsistency
- partial state with no rollback
- hard to reproduce and debug

Refactor target:
- pass `CancellationToken` only to reads and single atomic writes
- use `CancellationToken.None` for write calls in multi-step flows without a transaction
- wrap multi-step writes in a transaction if cancellation support is needed

## EF Core performance anti-patterns

## 28. Monolithic Where predicate with external variable checks

Symptoms:
- single `.Where()` with `||` / `&&` mixing C# variable checks and entity predicates
- `!string.IsNullOrWhiteSpace(x) && c.Field.Contains(x)` inside the expression tree

Why it hurts:
- EF Core generates bloated SQL with always-false branches
- unreadable and fragile LINQ
- poor SQL query plans

Refactor target:
- evaluate filter presence outside the expression tree
- chain `.Where()` for AND, `.Union()` for OR
- see `references/data-access-guidance.md` for full examples

## 29. Loading full entities for read-only endpoints

Symptoms:
- `.ToListAsync()` on entity set, then mapping to DTOs in C#
- no `.Select()` projection, no `.AsNoTracking()`

Why it hurts:
- fetches all columns including large ones
- change tracker allocates per-entity overhead

Refactor target:
- `.Select(x => new ResponseDto(...))` directly in the query
- `.AsNoTracking()` when projection is not possible

## 30. N+1 queries via lazy loading or loop fetching

Symptoms:
- related entities loaded inside a `foreach` loop
- lazy loading proxies enabled

Why it hurts:
- one query per iteration
- hard to spot without query logging

Refactor target:
- `.Include()` for eager loading
- `.AsSplitQuery()` when multiple collection navigations

## 31. Loading entities solely for delete or update

Symptoms:
- `FindAsync` + `Remove` for bulk deletes
- loading entities to set a property, then `SaveChangesAsync`

Why it hurts:
- unnecessary round-trip per entity

Refactor target:
- `ExecuteDeleteAsync()` / `ExecuteUpdateAsync()` (EF Core 7+)

## Change management anti-patterns

## 24. Big-bang rewrite

Symptoms:
- “let’s convert the whole codebase to slices this sprint”

Why it hurts:
- high risk
- behavior drift
- merge hell

Refactor target:
- one slice at a time

## 25. Endless refactoring with no acceptance boundary

Symptoms:
- architecture cleanup with no business value boundary
- no measurable end condition

Why it hurts:
- rabbit hole
- unbounded work

Refactor target:
- explicit incremental milestones
- preserve behavior first

## Review format suggestion

When reviewing a repo, report findings as:

1. **Observed smell**
2. **Why it matters**
3. **Smallest useful refactor**
4. **Target structure**
5. **Risk level**
6. **Suggested sequence**

Example:

```text
Observed smell:
  Orders feature is split across Controllers, Services, Repositories, Validators.

Why it matters:
  A single change to CancelOrder spans five folders and encourages service growth.

Smallest useful refactor:
  Move CancelOrder into Features/Orders/CancelOrder with its own request, validator, handler, and result mapping.

Risk level:
  Low if done endpoint-by-endpoint.

Suggested sequence:
  CancelOrder -> GetOrderById -> CreateOrder
```
