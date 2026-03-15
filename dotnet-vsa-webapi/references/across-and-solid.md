# ACROSS and SOLID in practice

This skill uses ACROSS as the main architecture mindset and SOLID as a supporting local design toolkit.

The goal is not textbook purity.
The goal is **safe, local, reversible change with low cognitive load**.

## ACROSS summary

- **A** — Abstractions & Decomposition
- **C** — Composition by Default
- **R** — Rabbit Hole avoidance
- **O** — Optimize for Change
- **S** — Simple As Possible
- **S** — Screaming Contract

## ACROSS -> concrete code-generation rules

## A — Abstractions & Decomposition

### Intent

Split code into parts with clear responsibilities and explicit contracts.

### Apply it like this

- decompose by feature first
- isolate infrastructure edges
- expose inter-module behavior through narrow contracts
- keep handlers focused on one use case
- use interfaces at meaningful seams only

### Good questions

- What changes together?
- What should not know about this implementation detail?
- What is the narrowest contract this caller really needs?

### Concrete rules

Do:
- create `IShipmentNumberGenerator` if numbering may vary
- create `IBillingModuleApi` for module-to-module calls
- create `IDbConnectionFactory` for Dapper access

Do not:
- create `IGetOrderByIdHandler` only because every class “should have an interface”
- create `IRepository<T>` to hide every query from EF Core

## C — Composition by Default

### Intent

Prefer assembling behavior from collaborators instead of inheritance trees.

### Apply it like this

- inject collaborators into handlers/services
- use small policies/strategies for variant behavior
- keep domain behavior in entities/value objects when it clarifies invariants

### Prefer

- strategy over template method
- helper/service composition over base endpoint class
- extension methods or static mapping functions over inheritance hierarchies

### Avoid

- `BaseCrudService<T>`
- `BaseEndpoint<TRequest, TResponse>`
- inheritance-heavy “frameworks” inside the app

## R — Rabbit Hole avoidance

### Intent

Avoid deep call stacks, over-factored flows, and endless refactors.

### Apply it like this

- keep request flow obvious
- avoid handler -> service -> orchestrator -> manager -> helper -> executor chains
- refactor in short, shippable increments
- be suspicious of “clever” indirection

### Concrete rules

A reader should be able to trace a request from endpoint to handler to data access in a small number of jumps.

Bad:

```text
Endpoint
 -> Facade
   -> ApplicationService
     -> DomainCoordinator
       -> UseCaseExecutor
         -> RepositoryAdapter
```

Better:

```text
Endpoint
 -> Handler
   -> DbContext / Gateway / Domain
```

## O — Optimize for Change

### Intent

Design so expected changes are local, safe, and reversible.

### Apply it like this

- keep business capabilities isolated
- use narrow module APIs
- avoid direct database coupling across modules
- use expand-contract migration patterns for data changes
- favor additive changes before destructive changes

### Concrete rules

When a change request arrives, ask:
- Which slice/module should absorb this?
- Can one team change it without coordinating five others?
- Can we roll this back without breaking everything?

### Examples

Good:
- swap payment provider by changing one gateway adapter and config

Bad:
- provider switch requires edits across endpoints, services, DTOs, and database code in multiple modules

## S — Simple As Possible

### Intent

Use the least complex structure that solves today’s problem well.

### Apply it like this

- keep logic local first
- extract after repeated same-reason duplication
- use direct mapping code instead of adding a mapper library
- use direct DbContext access in slices when repository abstraction adds no value

### Concrete rules

Choose the simplest thing that is:
- readable
- testable
- safe to change
- explicit

Do not choose the simplest thing that becomes a trap after one feature.

## S — Screaming Contract

### Intent

Names and contracts should tell the business story directly.

### Apply it like this

- use domain names, not technical placeholders
- return `Result`/`Error`, not `bool`
- expose meaningful endpoint names and operations
- name policies after business rules

### Prefer

- `OrderAlreadyPaid`
- `ReserveInventory`
- `GetCustomerBalance`
- `RequireAccountingAccess`

### Avoid

- `Process`
- `Execute`
- `Success = false`
- `UnknownError`

## How SOLID fits inside ACROSS

SOLID is useful, but not as dogma.

## SRP — Single Responsibility

Use SRP to keep handlers, policies, and abstractions cohesive.

Practical interpretation:
- one slice = one business request
- one handler = one workflow
- one interface = one cohesive boundary

Do not turn SRP into microscopic classes for sport.

## OCP — Open/Closed

Use only where variation is real.

Good uses:
- provider strategies
- pricing policies
- tax calculators
- notification dispatch strategies

Do not create extension points for imagined future needs.

## LSP — Liskov Substitution

Rarely a primary design driver in application code.
Relevant mostly when you intentionally create inheritance-based extension points.

In this architecture, composition usually matters more.

## ISP — Interface Segregation

Very important.

Use narrow interfaces:
- `ICurrentUser`
- `IDbConnectionFactory`
- `IInventoryGateway`

Do not lump unrelated operations into one “utility” interface.

## DIP — Dependency Inversion

Foundational.

Higher-level business logic should depend on stable contracts or infrastructure primitives that do not leak volatile implementation details.

Examples:
- handler depends on `IInventoryGateway`
- module depends on `IBillingModuleApi`
- query handler depends on `IDbConnectionFactory`

## Review heuristics

When reviewing code, ask:

### Decomposition
- Are the boundaries aligned with business change?
- Is this abstraction buying isolation or just ceremony?

### Composition
- Is inheritance used where composition would be simpler?

### Rabbit hole
- Can I follow the request flow without digging through five layers?

### Optimize for change
- Will this change stay local next time?

### Simplicity
- Is this the simplest design that still leaves room for evolution?

### Screaming contract
- Do names tell the domain story clearly?

## Common decisions

## When to introduce an interface

Introduce it when:
- the dependency is external or unstable
- multiple slices need the same narrow contract
- a module boundary is being established
- a stable seam improves tests and change isolation

Keep the concrete class when:
- it is local and stable
- the abstraction would only mirror one implementation with no real boundary value
- the caller already depends on the correct primitive (`DbContext`, `TimeProvider`, etc.)

## When to extract shared logic

Extract when:
- it appears in multiple slices
- it changes for the same reason
- extraction makes navigation easier

Do not extract when:
- the code only looks similar
- different slices will evolve independently
- the shared helper would become a junk drawer

## Short review checklist

- Is the design centered on change?
- Is the slice local and cohesive?
- Are abstractions meaningful?
- Is composition preferred?
- Is the flow shallow and debuggable?
- Are names business-first?
- Is the code simpler after this design, not just more “architected”?
