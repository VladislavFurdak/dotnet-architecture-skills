# Review Checklist: ASP.NET DDD Core Logic Skill

Use this checklist for self-review before returning the result.

## Modeling

- [ ] Bounded Context is defined explicitly
- [ ] Ubiquitous Language is consistent
- [ ] Aggregate Root is chosen intentionally
- [ ] Invariants live in the domain
- [ ] Value Objects are immutable
- [ ] Entity / Aggregate Root are not mutated directly

## Application layer

- [ ] The use case orchestrates rather than duplicating domain rules
- [ ] Repository interfaces live in `Application`
- [ ] Messaging / Outbox ports live in `Application`
- [ ] All async methods accept `CancellationToken`

## Infrastructure boundaries

- [ ] `Domain` does not depend on infrastructure
- [ ] `Infrastructure.Persistence` implements repository ports
- [ ] `Infrastructure.Messaging` implements publication and dispatcher logic
- [ ] The API does not contain domain logic

## Events

- [ ] Domain events are separated from integration events
- [ ] There is an explicit mapping from domain → integration event
- [ ] There is no direct broker publication after commit
- [ ] Transactional Outbox is used

## Rich domain model (class-based approach, when used)

- [ ] All mutable state uses private setters
- [ ] Aggregate creation is only through static factory methods
- [ ] Private parameterless constructor exists for EF Core
- [ ] Guard clauses validate preconditions in constructors and methods
- [ ] Result pattern used for expected business failures (not exceptions)
- [ ] Domain events collected in AggregateRoot base class
- [ ] EF Core maps value objects via `OwnsOne`, not ORM attributes in domain
- [ ] `DomainEvents` property is ignored in EF mapping
- [ ] EF interceptor publishes domain events; outbox handles integration events
- [ ] Factory methods raise initial domain event (e.g., `AccountCreated`)

## Anti-patterns

- [ ] No CQRS/CQS by default
- [ ] No Event Sourcing by default
- [ ] No generic repository<T> as the primary abstraction
- [ ] No EF/ORM annotations in the domain model
- [ ] No API DTO used instead of the domain model
- [ ] No public setters for ORM convenience
- [ ] No direct domain event publishing from repositories

## Output quality

- [ ] There is an explanation of the project structure
- [ ] There is code for Domain and Application
- [ ] It shows how the outbox is persisted
- [ ] It shows how a worker publishes integration events
- [ ] It proposes unit and integration tests
