# dotnet-aspnet-ddd-core-logic

A Claude Code skill for designing and implementing the **core business logic of ASP.NET web applications** using **Domain-Driven Design (DDD)**.

This skill is intended for tasks such as:

- designing aggregates and aggregate roots
- modeling entities and value objects
- implementing application use cases
- introducing domain events and integration events correctly
- structuring repository and messaging ports
- applying Transactional Outbox for reliable event publication
- keeping the domain model independent from storage, HTTP, ORM, and broker concerns

The skill is opinionated: it promotes a **rich domain model**, explicit aggregate boundaries, clear assembly dependencies, and reliable event delivery without adopting heavyweight patterns by default.

## What this skill optimizes for

This skill helps generate or review code that is:

- **domain-first** rather than infrastructure-first
- **behavior-rich** rather than CRUD-centric
- **testable without external resources**
- **consistent in language and naming** through Ubiquitous Language
- **safe for production event delivery** through Transactional Outbox
- **pragmatic** about DDD, instead of mechanically applying every possible pattern

It is especially useful when the goal is not just to "make the API work", but to preserve business rules, invariants, and long-term maintainability in the application core.

## Architectural style

The skill enforces the following assembly dependency structure:

- `Domain` → depends on nothing
- `Application` → depends on `Domain`
- `Infrastructure.Persistence` → depends on `Application` and `Domain`
- `Infrastructure.Messaging` → depends on `Application`
- `API` → depends on `Application`
- composition happens via DI in the entry point

### Interface placement

Repository ports and messaging/outbox ports live in **`Application`**, while their implementations live in infrastructure projects.

This is a deliberate rule:

- `Domain` stays isolated from storage and transport concerns
- `Application` defines the contracts required to execute use cases
- `Infrastructure` provides concrete implementations

## Core practices and approaches

### 1. Domain first

Every implementation starts from the business model:

- bounded context
- ubiquitous language
- aggregate boundaries
- invariants
- domain events
- integration events
- required data for the use case

The skill avoids starting from controllers, database schema, ORM entities, or broker contracts.

### 2. Rich domain model

Business rules that belong to the internal state of an aggregate or entity must live in the domain model.

The skill discourages an anemic model where domain objects are just mutable data containers and all business behavior is pushed into services, handlers, or controllers.

### 3. Immutable modeling by default

The default recommendation is:

- `Value Object` → immutable `record`
- `Entity` / `Aggregate Root` → immutable `sealed record` with explicit identity
- collections → immutable representations such as `ImmutableArray` or `IReadOnlyList`

Aggregate operations should return a new version of the aggregate instead of mutating public state directly.

### 4. Pragmatic class-based alternative

The skill also allows a **class-based rich domain model** when immutable records become impractical, for example:

- complex EF Core mapping
- aggregate state machines
- teams that prefer `Result`-style business outcomes
- use cases where controlled internal mutation is simpler and clearer

This is not treated as a fallback to bad design. It is accepted when invariants remain protected and the model stays behavior-rich.

Key patterns for the class-based approach:

- **Private setters** — all mutable state is protected
- **Static factory methods** — the only public way to create aggregates
- **Result pattern** — `Result` / `Result<T>` for expected business failures instead of exceptions
- **Guard clauses** — precondition validation in constructors and methods
- **AggregateRoot / Entity base classes** — identity equality and domain event collection
- **EF Core `OwnsOne`** — value object persistence without ORM attributes in domain
- **EF Core `SaveChangesInterceptor`** — automatic domain event publishing

### 5. Clear separation of responsibilities

The skill keeps responsibility boundaries explicit:

- **Domain** contains business rules and invariants
- **Application** orchestrates use cases
- **Infrastructure.Persistence** persists aggregate state and outbox records
- **Infrastructure.Messaging** publishes integration events from the outbox
- **API** handles transport contracts and delegates to a single use case

### 6. Transactional Outbox instead of direct publish-after-commit

The skill explicitly avoids publishing integration events directly after the database transaction commits.

Instead, it recommends **Transactional Outbox**:

1. complete the business operation
2. persist aggregate changes
3. store outbox records in the same local transaction
4. let a separate worker publish messages to the broker
5. mark outbox records as processed

This prevents the classic failure window where the database commit succeeds but the process crashes before the event is published.

### 7. Domain events and integration events are not the same

The skill makes a strict distinction:

- **Domain Events** describe facts inside the same bounded context and may trigger internal reactions
- **Integration Events** are contracts intended for inter-process, inter-service, or inter-context communication

Domain events are part of domain behavior.
Integration events are part of application/integration behavior.

When using the class-based approach, domain events can be published automatically via an EF Core interceptor, while integration events always go through the Transactional Outbox.

### 8. Ubiquitous Language is mandatory

Naming must remain stable inside the same bounded context.

If the context uses the term `Order`, the implementation should not randomly switch to `Purchase`, `CartOrder`, or `Request` without a domain reason. The same applies to file names, namespaces, event names, repository contracts, and use-case names.

## Supported DDD patterns

This skill supports both **Tactical DDD** and **Strategic DDD**.

### Tactical DDD patterns

#### Entity

Used for domain objects with identity that persists over time.
Entities are expected to contain behavior, not only data.

#### Value Object

Used for immutable domain concepts without identity.
Examples: `Money`, `Address`, `AccountNumber`.

#### Aggregate / Aggregate Root

Used to define consistency boundaries.
External code should access an aggregate only through its root.
Invariants inside the aggregate are enforced synchronously.

#### Repository

Repositories are abstractions for loading and saving **aggregate roots**, not every entity in the model.
The skill discourages "repository per table" or "generic repository for everything".

#### Factory

Factories are used when aggregate creation is non-trivial and must guarantee a valid initial state.
In the class-based approach, static factory methods on the aggregate root serve this role.

#### Domain Service

Used for business logic that does not naturally belong to a single entity or value object.

#### Domain Event

Used for explicit domain facts inside the bounded context.
Examples: `OrderPlaced`, `PaymentCaptured`, `InventoryReserved`.

#### Specification

Used when a business rule or selection rule benefits from being modeled explicitly and reused.

### Strategic DDD patterns

#### Bounded Context

The skill expects the domain model to be scoped to a clear context. It avoids mixing multiple subdomains into a single ambiguous language model.

#### Context Map

When multiple contexts are involved, the relationship between them should be explicit rather than implied.

#### Anti-Corruption Layer

When integrating with legacy or external systems, the skill prefers a translation layer so that external terminology and contracts do not leak into the internal domain model.

## Recommended implementation flow

When generating a feature, the skill follows this order:

1. define the **Ubiquitous Language**
2. define the **Bounded Context**
3. model the **Aggregate Design**
4. identify **Domain Events** and **Integration Events**
5. define the **Application Flow**
6. map the **Project/File Structure**
7. produce the **Code**
8. explain **Outbox Notes**
9. suggest **Tests**

This flow keeps code generation aligned with the domain model instead of letting infrastructure drive design decisions.

## What the skill encourages

The skill encourages:

- aggregate-oriented design
- explicit invariants
- rich domain behavior
- use-case orchestration in `Application`
- repository and messaging ports in `Application`
- immutable value objects
- explicit domain types instead of primitive obsession
- Transactional Outbox for integration events
- domain-event-based internal reactions
- testing the domain without a database or broker
- vertical slices by use case inside `Application`
- Minimal APIs or Controllers based on project style, not ideology

## Anti-patterns this skill actively avoids

This skill is intentionally defensive against common architecture mistakes.

### 1. Anemic domain model

Avoid turning entities into passive data bags while placing all business logic into handlers, application services, or controllers.

### 2. Infrastructure leakage into the domain

Avoid bringing these into `Domain`:

- `DbContext`
- ORM attributes
- HTTP DTOs
- broker client types
- persistence-specific annotations
- storage-specific concerns

### 3. Generic repository everywhere

Avoid using one generic repository abstraction for all entities.
Repositories should model aggregate persistence, not hide the database behind a universal CRUD interface.

### 4. CQRS/CQS as a default reflex

The skill does **not** assume CQRS or command/query splitting by default.
It only becomes appropriate when there is a concrete reason.

### 5. Event Sourcing as a default reflex

The skill does **not** assume Event Sourcing by default.
The current aggregate state in storage remains the source of truth unless there is a strong business need for something else.

### 6. Direct publish after commit

Avoid the unreliable flow:

1. commit business data
2. publish integration event directly to Kafka/RabbitMQ/etc.

That design introduces an event-loss window.
The skill recommends outbox-based publication instead.

### 7. Controller-driven domain logic

Avoid placing business rules in controllers, endpoints, or request handlers.
The API layer should validate transport contracts and delegate to one use case.

### 8. Mutable entities with public setters

Avoid mutable domain models with unrestricted public setters and delayed validation.
State transitions should be controlled by domain behavior.

### 9. Reusing API DTOs as domain models

Avoid making transport models double as domain models.
The domain should keep its own types, language, and invariants.

### 10. CRUD-first architecture disguised as DDD

Avoid mapping tables first and then calling the result "DDD".
If the model has no invariants, no aggregate boundaries, and no meaningful behavior, the design is not domain-driven.

## Reliability model for events

The skill uses a two-level event model:

### Domain event flow

- raised by aggregates
- used inside the same bounded context
- may trigger in-process side effects
- may be dispatched during save via an EF Core interceptor (class-based approach)
- or accumulated as pending events in the aggregate (immutable records approach)

### Integration event flow

- created from application/domain outcomes
- serialized into outbox records
- persisted in the same transaction as business data
- published by a dedicated worker in `Infrastructure.Messaging`
- retried and tracked independently from the request flow

This split keeps internal domain reactions fast and explicit, while cross-system communication remains reliable.

## ASP.NET guidance

When used in ASP.NET applications, the skill prefers the following style:

- keep `Application` organized by use case or vertical slice
- do not force CQRS naming if the project does not need it
- choose **Minimal API** or **Controllers** based on the existing codebase style
- compose dependencies in the entry point
- register persistence and messaging adapters through dedicated extension methods

## File structure in this skill

This folder contains the core skill file and supporting material:

- `SKILL.md` — the main skill definition and implementation guidance
- `templates/use-case-template.md` — a reusable template for use-case design and implementation
- `examples/order-placement-sample.md` — example with immutable records approach
- `examples/bank-account-rich-model-sample.md` — example with class-based rich model approach
- `references/review-checklist.md` — review checklist for validating generated solutions

## When to use this skill

Use this skill when the task is about:

- domain modeling for an ASP.NET web app
- building or refactoring business logic
- defining aggregates and value objects
- designing application services and use cases
- introducing repositories, domain events, or integration events correctly
- aligning code with DDD practices without unnecessary architectural complexity

## When not to use it as-is

This skill may be excessive for:

- very small CRUD-only admin panels
- temporary prototypes with no meaningful business rules
- read-heavy/reporting-only features where the domain model is not the primary concern

Even in those cases, some parts of the guidance may still be useful, especially around boundaries, naming, and avoiding infrastructure leakage.

## Summary

`dotnet-aspnet-ddd-core-logic` is a practical DDD skill for ASP.NET teams who want a **clean domain model**, **clear responsibility boundaries**, and **reliable integration event handling**.

It combines:

- tactical DDD patterns
- strategic DDD boundaries
- application-layer orchestration
- infrastructure isolation
- Transactional Outbox reliability
- explicit anti-pattern avoidance
- two modeling approaches: immutable records and class-based rich model

The result is a skill designed to help generate solutions that are easier to evolve, test, and reason about in real production systems.
