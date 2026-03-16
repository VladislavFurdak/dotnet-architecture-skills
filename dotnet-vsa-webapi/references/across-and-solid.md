# ACROSS and SOLID in practice

**ACROSS is not an industry-standard acronym.** It is a practical architecture decision framework created by Vladyslav Furdak that distills six principles for designing maintainable application code. This skill uses ACROSS as the primary architecture mindset and SOLID as a supporting local design toolkit.

### Why ACROSS instead of SOLID as the primary lens

SOLID principles (SRP, OCP, LSP, ISP, DIP) move us toward the same goal — localizing change — but they can become a source of unnecessary abstractions and rituals:
- **SRP**: "single responsibility" is philosophical without context — responsibility depends on the size of the context you apply it to
- **OCP**: "extend, don't modify" requires additional abstractions that may add more technical debt than benefit
- **LSP**: rarely a primary driver in application code unless you're writing your own framework
- **ISP**: a direct consequence of SRP applied to interfaces — useful but derivative
- **DIP**: foundational and practical — but it is a special case of **A** (Abstractions & Decomposition) from ACROSS

ACROSS doesn't reject SOLID/GRASP, but **removes dogmatism and puts change at the center of design**. A program doesn't have to look "beautiful" or "logical" — it has to be flexible without creating extra problems when changes are made, and easy to test with automated tests.

The goal is not textbook purity.
The goal is **safe, local, reversible change with low cognitive load**.

## ACROSS summary

Each letter addresses a specific architecture question:

- **A — Abstractions & Decomposition**: How do I split the system into parts? Create interfaces and boundaries only at meaningful seams — external gateways, module APIs, unstable dependencies. Avoid "interface per class" ceremony.
- **C — Composition by Default**: How do I assemble behavior? Inject collaborators and compose small strategies instead of building inheritance trees. No `BaseCrudService<T>`, no `BaseEndpoint`.
- **R — Rabbit Hole avoidance**: How deep is the call stack? Keep request flow traceable in 2-3 jumps (Endpoint -> Handler -> Data/Domain). Avoid handler -> service -> orchestrator -> manager -> helper chains.
- **O — Optimize for Change**: Will the next change stay local? Isolate business capabilities so a feature change doesn't ripple across modules. Favor additive changes, expand-contract migrations, and narrow module APIs.
- **S — Simple As Possible**: Am I over-engineering? Use the least complex structure that solves today's problem. Extract shared code only after repeated, same-reason duplication. Direct `DbContext` usage beats generic repository when the abstraction adds no value.
- **S — Screaming Contract**: Do names tell the business story? Use `OrderAlreadyPaid`, `ReserveInventory`, `RequireAccountingAccess` — not `Process`, `Execute`, `Success = false`. Return `Result`/`Error`, not `bool`.

## ACROSS -> concrete code-generation rules

## A — Abstractions & Decomposition

### Intent

Break the implementation into parts with defined responsibilities and build explicit abstractions between them (public methods, interfaces, facades, events). This is the **primary, central design principle** of ACROSS. Unlike SRP, we split into parts where each has *some* responsibility — not necessarily a single one — and we build a formal contract between these parts. Any architecture style (N-layered, Hexagonal, Clean, VSA) is built on varying levels of this decomposition.

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

For code reuse, apply composition first. Use inheritance only when you need to lock down the design of extensions and constraints. **Inheritance is the strongest form of coupling** — it is a tool for freezing a design, not for reusing code.

When to use inheritance: when building a class hierarchy that sets boundaries for descendants and governs extension (e.g., .NET's `Stream`, ADO.NET's `DbCommand`/`DbConnection` — these are infrastructure frameworks). Business logic is more convenient and flexible when written with composition and a functional style.

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
- premature base classes
- using Template Method where a Strategy would suffice
- inheritance-heavy “frameworks” inside the app

## R — Escape from the Rabbit Hole

### Intent

Avoid the “rabbit hole” — deep call stacks within a single layer and endless, goal-less refactoring. Work in short iterations: release -> feedback -> continue.

Two meanings of “rabbit hole”:
1. **Deep nesting**: method calling method calling method 20 times until you lose context. Ideal: methods of at most a few hundred lines, without uncontrolled plunges into dozens of other methods within the same layer.
2. **Endless refactoring**: taking on huge tasks all at once — spending weeks on a refactor. Define reasons and metrics (limited scope, % test coverage) before starting changes.

### Apply it like this

- keep request flow obvious
- avoid handler -> service -> orchestrator -> manager -> helper -> executor chains
- refactor in short, shippable increments — not “rewrite everything” sprints
- be suspicious of “clever” indirection
- agree on refactoring metrics before starting changes

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

Design the system so that any anticipated change is **local, safe, and reversible** — with minimal coordination and no cascading side effects. There is always a backup plan and a way to roll back.

### Apply it like this

- keep business capabilities isolated
- use narrow module APIs
- avoid direct database coupling across modules
- use **expand-contract migration patterns** for data changes
- favor additive changes before destructive changes

### Concrete rules

When a change request arrives, ask:
- Which slice/module should absorb this?
- Can one team change it without coordinating five others? (minimal coordination)
- Can we roll this back without breaking everything?

### Examples

**Good — minimal coordination:**
- Replacing a payment provider: change one `IPaymentGateway` adapter and flip a feature flag — only the payments team releases.

**Bad — cascading side effects:**
- New provider requires changes across five services, a coordinated release, and a sprint of negotiations.
- A column is renamed in the DB -> seven services crash because they read it directly.

### Expand-contract pattern

When renaming or migrating data, use double-write during the transition:

```csharp
// Read: prefer new field, fall back to old
public static string Read(UserRow r) => r.DisplayName ?? r.FullName ?? "";

// Write: write both during migration
public static void Write(UserRow r, string displayName)
{
    r.DisplayName = displayName;
    r.FullName ??= displayName; // double-write during migration
}
```

Add the new field. Read from new ?? old. Write to both. No consumers break. Remove the old field after all consumers migrate.

## S — Simple As Possible

### Intent

Implement things as simply as today’s requirements allow. **Generalize only what is duplicated at least three times.** Create additional abstractions only if you’re confident they will save development time or reduce errors. Comparable to KISS and YAGNI, but SAP calls for the **maximum feasible simplicity** rather than leaving the target degree of complexity undefined.

### Apply it like this

- keep logic local first
- extract after repeated same-reason duplication (rule of three)
- use direct mapping code instead of adding a mapper library
- use direct DbContext access in slices when repository abstraction adds no value
- if building a simple CRUD service, N-layered or direct `DbContext` from Minimal API is acceptable — no need for Clean Architecture abstractions

### Concrete rules

Choose the simplest thing that is:
- readable
- testable
- safe to change
- explicit

Do not choose the simplest thing that becomes a trap after one feature. Before building abstractions, consider whether they solve a real unification problem or just look like deduplication.

## S — Screaming Contract

### Intent

Any entities or methods should, by their names, tell you **what they are and how to use them** — contracts that scream the domain. This approach is most aligned with DDD naming, but you can use it without tying yourself to DDD patterns.

### Apply it like this

- **Names = the actual action**, not the mechanics: `ReserveInventory`, `CapturePayment`, not `Process()`
- **Explicit operation outcome**: use `Result`/`Error` types instead of `bool` or generic exceptions
- **API speaks the domain**: `POST /orders/{id}/payment/capture` — follow RPC-style naming for operations, REST for content/resource manipulation
- **Events speak the domain**: `InvoicePaid`, not `PaymentOperationCompleted`
- **A cancellation is `CancelBooking`**, not `Update(status = Cancelled)`
- **An error is `PaymentDeclined`**, not `500 / "Unknown"`
- The provider's contract is hidden; externally, expose pure domain language

### Prefer

- `OrderAlreadyPaid`
- `ReserveInventory`
- `CapturePayment`
- `GetCustomerBalance`
- `RequireAccountingAccess`

### Avoid

- `Process`
- `Execute`
- `Handle` without domain context
- `Success = false`
- `UnknownError`

## How SOLID fits inside ACROSS

SOLID is useful, but not as dogma. Any experienced developer eventually arrives at similar practical skills, but SOLID is very local in scope and shouldn't be applied blindly — it always depends on context. In practice, SRP and DIP are the most daily-relevant; the others are situational.

## SRP — Single Responsibility

Use SRP to keep handlers, policies, and abstractions cohesive.

Practical interpretation:
- one slice = one business request
- one handler = one workflow
- one interface = one cohesive boundary

Do not turn SRP into microscopic classes for sport.

## OCP — Open/Closed

Use only where variation is real. OCP sometimes nudges toward an artificial "extend-only" approach — decide whether following it brings more benefit than technical debt.

Good uses:
- provider strategies
- pricing policies
- tax calculators (real example: `ITaxCalculator` with `EuVatCalculator`, `UsSalesTaxCalculator`)
- notification dispatch strategies

```csharp
public interface ITaxCalculator
{
    Money Calculate(Money net, CountryCode shipTo);
}

public class EuVatCalculator : ITaxCalculator { /* ... */ }
public class UsSalesTaxCalculator : ITaxCalculator { /* ... */ }
```

This abstraction is justified by OFC (Optimize for Change) — adding a new country's tax rules is local: add one class, register it.

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
