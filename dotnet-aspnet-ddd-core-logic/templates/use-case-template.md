# DDD Use Case Template for ASP.NET Core Logic

## 1. Ubiquitous Language

- Bounded Context:
- Aggregate Root:
- Entities:
- Value Objects:
- Domain Service:
- Domain Events:
- Integration Events:

## 2. Business Goal

Describe the business goal of the use case in one paragraph.

## 3. Invariants

- (state-based rule, e.g., "Order can only be placed from Draft status")
- (data constraint, e.g., "Balance cannot go below zero")
- (cross-field rule, e.g., "Currency must match between amount and account")

## 4. Aggregate Design

Modeling approach: [ ] Immutable records  [ ] Class-based rich model

### Aggregate Root

```csharp
// aggregate root code
// If class-based: private setters, static factory, Guard clauses, Result returns
// If immutable records: sealed record, methods return new instance
```

### Value Objects

```csharp
// value object code
// Constructor validation, factory methods, formatting helpers
// If class-based with EF Core: ensure private parameterless constructor for ORM
```

### Domain Events

```csharp
// domain event code
// If class-based: collected via AddDomainEvent() in base class
// If immutable records: accumulated in ImmutableArray<IDomainEvent>
```

## 5. Application Flow

1. Load the required aggregates through repository ports.
2. Execute the business operation through the aggregate/domain service.
3. Obtain the final state of the aggregates.
4. Persist the aggregates.
5. Build the integration events.
6. Persist outbox entries in the same transaction.
7. Return the use-case result.

## 6. Application Ports

```csharp
// repositories, transaction boundary, outbox writer, serializers, clock
```

## 7. Use Case Implementation

```csharp
// application use case code
```

## 8. Infrastructure.Persistence Notes

- How the aggregate ↔ storage model mapping works
- How atomicity of aggregate updates + outbox is ensured
- Where the transaction boundary lives
- If class-based: EF Core `OwnsOne` configuration for value objects, `Ignore` for DomainEvents
- If class-based: EF Core interceptor setup for domain event publishing

## 9. Infrastructure.Messaging Notes

- How the worker reads the outbox
- How integration events are published
- How retry / idempotency / poison messages are handled

## 10. API Entry Point

```csharp
// minimal api or controller example
```

## 11. File Structure

```text
src/
  <Context>.Domain/
    <AggregateRoot>/
      <AggregateRoot>.cs
      <ValueObject>.cs
      <DomainEvent>.cs
      <DomainService>.cs          (if needed)
    Common/
      Entity.cs                   (if class-based)
      AggregateRoot.cs            (if class-based)
      IDomainEvent.cs
      Guard.cs                    (if class-based)
      Result.cs                   (if class-based)
      DomainException.cs

  <Context>.Application/
    Abstractions/
      Persistence/
        I<AggregateRoot>Repository.cs
        IApplicationTransaction.cs
        IOutboxWriter.cs
      Services/
        ISystemClock.cs
        IIntegrationEventSerializer.cs
    IntegrationEvents/
      IIntegrationEvent.cs
      <Name>IntegrationEvent.cs
    Outbox/
      OutboxMessage.cs
    UseCases/
      <UseCaseName>/
        <UseCaseName>Command.cs
        <UseCaseName>UseCase.cs

  <Context>.Infrastructure.Persistence/
    <AggregateRoot>/
      Ef<AggregateRoot>Repository.cs
    Outbox/
      EfOutboxWriter.cs
    Transactions/
      EfApplicationTransaction.cs
    Configuration/
      <AggregateRoot>Configuration.cs  (EF Core entity config)
    Interceptors/
      DomainEventPublishingInterceptor.cs  (if class-based)

  <Context>.Infrastructure.Messaging/
    Dispatching/
      OutboxDispatcherBackgroundService.cs
    Publishing/
      BrokerIntegrationEventPublisher.cs

  <Context>.API/
    Endpoints/
      <AggregateRoot>Endpoints.cs
```

## 12. Tests

### Unit tests

- aggregate invariants (status transitions, data constraints)
- domain service rules
- event emission (correct events raised after operations)
- If class-based:
  - Result pattern: success and failure paths for each business operation
  - Guard clauses: precondition violations throw on invalid arguments
  - Factory methods: correct initial state and initial domain event
  - Value object validation: constructor rejects invalid input

### Integration tests

- repository persistence (save and load round-trip preserves aggregate state)
- transaction writes domain state + outbox atomically
- outbox dispatcher publishes and marks processed
- If class-based with interceptor: domain events published on SaveChanges
