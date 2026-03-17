---
name: aspnet-ddd-core-logic
description: Designs and implements core business logic for ASP.NET web apps using Tactical and Strategic DDD. Use when user asks to "design an aggregate", "implement a use case", "model a bounded context", "create domain entities", "add domain events", "implement repository pattern", or build features with rich domain models, value objects, and Transactional Outbox. Does not use CQRS or Event Sourcing by default.
metadata:
  author: Vladyslav Furdak
  version: 2.0.0
  argument-hint: feature description, use case, or bounded context
---

# Purpose

Use this skill whenever you need to design or implement the **core business logic of an ASP.NET web app** using DDD practices.

This skill defines a **reference implementation style**, not just a list of patterns. Its goal is to keep the domain model clean, consistent, and independent from infrastructure.

# Core Rules

1. **Domain first, infrastructure second.** Every solution starts with Ubiquitous Language, bounded context, invariants, and aggregate boundaries.
2. **Do not use CQRS/CQS by default.** Split commands and queries only when there is a clear reason, not as a template.
3. **Do not use Event Sourcing by default.** The source of truth is the current aggregate state in storage.
4. **Ubiquitous Language is mandatory.** If the context uses the term `Order`, do not switch to `Purchase`, `Request`, or `CartOrder` without an explicit reason.
5. **All domain models are immutable.** Prefer `sealed record` and immutable collections by default. Do not mutate object state directly; create a new version via `with`, copying, or a factory method.
6. **Do not allow an anemic domain model.** If a rule belongs to the internal state of an entity or aggregate, it must live in the domain.
7. **Do not mix domain events and integration events.** They belong to different responsibility levels.
8. **Do not publish integration events directly after commit.** For inter-process communication, always prefer **Transactional Outbox**.

# Target Assembly Structure

Follow this dependency structure:

- `Domain` → depends on nothing
- `Application` → depends on `Domain`
- `Infrastructure.Persistence` → depends on `Application` and `Domain`
- `Infrastructure.Messaging` → depends on `Application`
- `API` → depends on `Application`
- composition through DI — in the entry point

# Where Interfaces Live

The explicit project rule takes priority:

- **Repository ports** and **Messaging/Outbox ports** live in `Application`
- implementations live in `Infrastructure.Persistence` and `Infrastructure.Messaging`
- `Domain` must not know infrastructure details or concrete storage/delivery mechanisms

If the task says that data-access interfaces should live “next to aggregates,” interpret that as a requirement to **keep infrastructure out of the domain**, while the **physical placement of interfaces stays in `Application`**, because that matches the assembly dependency rules.

# Mandatory Design Process

Whenever you implement a feature, always move in this order:

## 1) Establish the bounded context and Ubiquitous Language

Before writing code, explicitly define:

- bounded context
- aggregate root
- entities
- value objects
- domain invariants
- domain events
- integration events
- what data is required for the use case

If the user did not provide a bounded-context name, choose a clean and consistent one yourself and keep using it.

## 2) Model the aggregates

Modeling rules:

- an aggregate is a consistency boundary
- external code interacts only through the aggregate root
- invariants inside the aggregate are enforced synchronously
- cross-aggregate reactions happen through domain events or eventual consistency
- do not create a repository for every entity; repositories are primarily for aggregate roots

By default:

- `Value Object` → immutable `record`
- `Entity` / `Aggregate Root` → immutable `sealed record` with explicit identity
- collections → `ImmutableArray<T>` / `IReadOnlyList<T>` / other immutable representations

Aggregate methods **do not mutate the current instance**. They return a new aggregate instance, for example:

- `Order Place(...) => new state`
- `Order AddItem(...) => new state`
- or `OperationResult<Order>` / `(Order Aggregate, ImmutableArray<IDomainEvent> Events)`

If the domain needs to accumulate events, store them as pending events inside the aggregate snapshot or return them together with the new aggregate state. The key requirement is: do not lose them between business-operation execution and persistence.

## 3) Put business logic in the right place

Use this rule:

- if a rule belongs to the **internal state** of an aggregate/entity → the code goes into the domain
- if an operation coordinates **multiple aggregates**, external policies, or ports → the code goes into an `Application` service / use case
- if the logic is domain logic but does not naturally belong to one entity → use a **Domain Service**

Do not turn `Application` into the place where all business rules live. `Application` orchestrates; it does not replace the domain.

## 4) Collect domain events and integration events correctly

Separate events like this:

### Domain Events

Used for reactions **inside the bounded context**.
Examples:

- `OrderPlaced`
- `PaymentCaptured`
- `InventoryReserved`

They express a domain fact and may trigger additional logic in the same context.

### Integration Events

Used for communication **between processes / services / bounded contexts**.

Examples:

- `OrderPlacedIntegrationEvent`
- `PaymentCapturedIntegrationEvent`

Do not publish them directly from the aggregate to the broker.

## 5) Persist final state through repositories + outbox in one transaction

After the business operation completes:

- you have the final state of one or more aggregates
- persist aggregates through repository ports
- write **outbox** records in the same local transaction
- only then consider the use case successful

This is a mandatory reliability rule. Do not propose this flow:

1. commit business data
2. then publish to the broker

That flow is risky because there is an event-loss window between commit and publish.

## 6) Publish integration events from a separate worker

`Infrastructure.Messaging` must read the outbox and publish events through a concrete adapter.

Required model:

- `Application` contains the port for outbox / publication work
- `Infrastructure.Persistence` stores outbox records in the same transaction as aggregate changes
- `Infrastructure.Messaging` reads the outbox in a separate process/worker, sends to the broker, and marks the record as processed

# Architectural Constraints

Always follow these constraints:

- do not use EF Core entity types as domain models
- do not pull `DbContext`, broker clients, HTTP DTOs, or ORM attributes into `Domain`
- do not create public settable properties just for ORM convenience
- do not use a generic repository “for everything” instead of aggregate repositories
- do not move invariants out of aggregates into controllers or handlers
- do not replace the domain model with CRUD tables that have no behavior
- do not mix read-model optimizations into the domain model unless there is a clear need

# Tactical DDD: What to Support

Use these patterns when needed:

- `Entity`
- `Value Object`
- `Aggregate / Aggregate Root`
- `Repository`
- `Factory`
- `Domain Service`
- `Domain Event`
- `Specification`

But apply them intentionally, not mechanically.

# Strategic DDD: What to Support

If the task touches multiple areas, explicitly define:

- `Bounded Context`
- `Context Map`
- `Anti-Corruption Layer`

If there is an integration with a legacy or external system, always isolate the external contract from the internal domain language via an ACL.

# Canonical Implementation Shape

When generating a new feature, use this structure as the default.

## Domain

Place the following in `Domain`:

- aggregate roots
- entities
- value objects
- domain services
- domain events
- domain errors / invariants
- factories, when creation is non-trivial

The domain should look like something that can be tested without a database, broker, or web framework.

### Example domain-model style

```csharp
namespace Sales.Domain.Orders;

public sealed record OrderId(Guid Value);
public sealed record CustomerId(Guid Value);
public sealed record ProductId(Guid Value);

public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new DomainRuleViolation("Currencies must match.");

        return this with { Amount = Amount + other.Amount };
    }
}

public sealed record OrderItem(ProductId ProductId, int Quantity, Money UnitPrice)
{
    public Money LineTotal => new(UnitPrice.Amount * Quantity, UnitPrice.Currency);

    public static OrderItem Create(ProductId productId, int quantity, Money unitPrice)
    {
        if (quantity <= 0)
            throw new DomainRuleViolation("Quantity must be greater than zero.");

        return new OrderItem(productId, quantity, unitPrice);
    }
}

public enum OrderStatus
{
    Draft,
    Placed,
    Cancelled
}

public interface IDomainEvent
{
    DateTime OccurredOnUtc { get; }
}

public sealed record OrderPlaced(
    OrderId OrderId,
    CustomerId CustomerId,
    DateTime OccurredOnUtc) : IDomainEvent;

public sealed record Order(
    OrderId Id,
    CustomerId CustomerId,
    OrderStatus Status,
    ImmutableArray<OrderItem> Items,
    ImmutableArray<IDomainEvent> PendingDomainEvents)
{
    public static Order CreateDraft(OrderId id, CustomerId customerId) =>
        new(id, customerId, OrderStatus.Draft, ImmutableArray<OrderItem>.Empty, ImmutableArray<IDomainEvent>.Empty);

    public Order AddItem(ProductId productId, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainRuleViolation("Only a draft order can be changed.");

        var item = OrderItem.Create(productId, quantity, unitPrice);
        return this with { Items = Items.Add(item) };
    }

    public Order Place(DateTime nowUtc)
    {
        if (Status != OrderStatus.Draft)
            throw new DomainRuleViolation("Only a draft order can be placed.");

        if (Items.IsDefaultOrEmpty)
            throw new DomainRuleViolation("Order must contain at least one item.");

        return this with
        {
            Status = OrderStatus.Placed,
            PendingDomainEvents = PendingDomainEvents.Add(new OrderPlaced(Id, CustomerId, nowUtc))
        };
    }

    public Order ClearPendingDomainEvents() =>
        this with { PendingDomainEvents = ImmutableArray<IDomainEvent>.Empty };
}
```

Small deviations from this template are allowed, but only if they preserve immutability and invariants.

### Alternative: Class-based rich domain model

When the project uses EF Core with complex mapping or the team prefers a class-based approach, aggregates may use **classes with private setters** instead of immutable records. This is acceptable as long as invariant protection is maintained.

Choose this style when:

- EF Core change tracking is used and mapping immutable records to owned types is impractical
- the aggregate has complex state transitions (state machines)
- the team prefers `Result<T>` return types over exceptions for business rule violations
- aggregate methods modify internal state rather than returning new instances

Choose immutable records when:

- the aggregate is small and functional style fits naturally
- there is no ORM mapping overhead
- the team favors explicit state transitions via `with` expressions

#### Result pattern for domain operations

When using the class-based style, aggregate methods should return `Result` or `Result<T>` to communicate success or failure without throwing exceptions for expected business rule violations. Exceptions remain appropriate for programming errors and broken preconditions (via Guard clauses).

```csharp
public Result Activate()
{
    if (Status == AccountStatus.Active)
        return Result.Failure("Account is already active");

    if (Status == AccountStatus.Closed)
        return Result.Failure("Cannot activate a closed account");

    Status = AccountStatus.Active;
    AddDomainEvent(new AccountActivatedEvent(Id, CustomerId, DateTime.UtcNow));
    return Result.Success();
}

public Result<Transaction> Withdraw(Money amount, string description, string reference)
{
    if (Status != AccountStatus.Active)
        return Result<Transaction>.Failure("Account must be active to process withdrawals");

    if (Balance.Amount < amount.Amount)
        return Result<Transaction>.Failure("Insufficient funds");

    Balance = Balance.Subtract(amount);
    var transaction = Transaction.CreateWithdrawal(Id, amount, description, reference);
    AddDomainEvent(new WithdrawalProcessedEvent(Id, amount, Balance, transaction.Id));
    return Result<Transaction>.Success(transaction);
}
```

Use the Result pattern for operations that may fail due to business rules (account already closed, insufficient funds). Use Guard clauses and exceptions for preconditions that should never fail in correctly written code (null arguments, empty GUIDs).

#### Guard clauses

Use a `Guard` utility for constructor and method precondition validation:

```csharp
private Account(AccountNumber accountNumber, Guid customerId, AccountType type, Currency currency)
{
    Guard.AgainstNullOrEmpty(accountNumber, nameof(accountNumber));
    Guard.AgainstNullOrEmpty(currency, nameof(currency));
    Guard.AgainstEmptyGuid(customerId, nameof(customerId));

    Id = Guid.NewGuid();
    AccountNumber = accountNumber;
    CustomerId = customerId;
    Type = type;
    Balance = Money.Zero(currency);
    Status = AccountStatus.PendingActivation;
}
```

#### Factory methods on aggregate roots

Use static factory methods as the only public way to create aggregates. This ensures all invariants are satisfied and the initial domain event is raised:

```csharp
public class Account : AggregateRoot<Guid>
{
    // Private constructor — enforces creation through factory
    private Account() { }

    public static Account CreateNew(Guid customerId, AccountType type, Currency currency)
    {
        var accountNumber = AccountNumber.Generate();
        var account = new Account(accountNumber, customerId, type, currency);

        account.AddDomainEvent(
            new AccountCreatedEvent(account.Id, customerId, account.AccountNumber, type.ToString(), DateTime.UtcNow));

        return account;
    }
}
```

#### AggregateRoot and Entity base classes

When using the class-based approach, use base classes to handle identity equality and domain event collection:

```csharp
public abstract class Entity<TId>
{
    public TId Id { get; protected set; } = default!;

    public override bool Equals(object? obj)
    {
        if (obj is not Entity<TId> entity || GetType() != entity.GetType())
            return false;
        return EqualityComparer<TId>.Default.Equals(Id, entity.Id);
    }

    public override int GetHashCode() =>
        EqualityComparer<TId>.Default.GetHashCode(Id);
}

public abstract class AggregateRoot<TId> : Entity<TId>
{
    private readonly List<IDomainEvent> _domainEvents = [];

    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent domainEvent) => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

#### Rich value objects (class-based)

Value objects in the class-based model use `record` types with constructor validation, factory methods, formatting, and implicit conversions:

```csharp
public record Money
{
    public decimal Amount { get; init; }
    public Currency Currency { get; init; }

    public Money(decimal amount, Currency currency)
    {
        ArgumentNullException.ThrowIfNull(currency);
        if (decimal.Round(amount, currency.DecimalPlaces) != amount)
            throw new DomainException($"Amount has too many decimal places for currency {currency.Code}");
        Amount = amount;
        Currency = currency;
    }

    public static Money Zero(Currency currency) => new(0, currency);
    public Money Add(Money other) { /* currency validation + arithmetic */ }
    public Money Subtract(Money other) { /* currency validation + arithmetic */ }
    public Money Multiply(decimal factor) => new(Amount * factor, Currency);
    public bool IsZero => Amount == 0;
    public bool IsPositive => Amount > 0;
    public bool IsNegative => Amount < 0;
}

public record AccountNumber
{
    public string Value { get; }

    public AccountNumber(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new DomainException("Account number cannot be null or empty");
        if (!IsValidFormat(value.Trim()))
            throw new DomainException($"Invalid account number format: {value}");
        Value = value.Trim();
    }

    public static AccountNumber Generate() { /* business generation logic */ }
    public string ToFormattedString() => $"{Value[..3]}-{Value[3..6]}-{Value[6..]}";
    public static implicit operator string(AccountNumber an) => an.Value;
}
```

## Application

Place the following in `Application`:

- use cases / application services
- repository ports
- unit of work / transactional boundary abstractions, if needed
- outbox ports
- integration event contracts
- mapping from domain events to integration events

### Ports

```csharp
namespace Sales.Application.Abstractions.Persistence;

public interface IOrderRepository
{
    Task<Order?> GetById(OrderId orderId, CancellationToken ct);
    Task Save(Order order, CancellationToken ct);
}

public interface IOutboxWriter
{
    Task Add(OutboxMessage message, CancellationToken ct);
}

public interface IApplicationTransaction
{
    Task Execute(
        Func<CancellationToken, Task> action,
        CancellationToken ct);
}
```

### Integration Event Contract

```csharp
namespace Sales.Application.IntegrationEvents;

public interface IIntegrationEvent
{
    Guid EventId { get; }
    DateTime OccurredOnUtc { get; }
}

public sealed record OrderPlacedIntegrationEvent(
    Guid EventId,
    Guid OrderId,
    Guid CustomerId,
    DateTime OccurredOnUtc) : IIntegrationEvent;
```

### Outbox Model

```csharp
namespace Sales.Application.Outbox;

public sealed record OutboxMessage(
    Guid Id,
    string Type,
    string Payload,
    DateTime OccurredOnUtc,
    DateTime? ProcessedOnUtc,
    string? Error);
```

### Use Case

```csharp
namespace Sales.Application.UseCases.PlaceOrder;

public sealed record PlaceOrderCommand(OrderId OrderId);

public sealed class PlaceOrderUseCase
{
    private readonly IOrderRepository _orders;
    private readonly IOutboxWriter _outbox;
    private readonly IApplicationTransaction _transaction;
    private readonly ISystemClock _clock;
    private readonly IIntegrationEventSerializer _serializer;

    public PlaceOrderUseCase(
        IOrderRepository orders,
        IOutboxWriter outbox,
        IApplicationTransaction transaction,
        ISystemClock clock,
        IIntegrationEventSerializer serializer)
    {
        _orders = orders;
        _outbox = outbox;
        _transaction = transaction;
        _clock = clock;
        _serializer = serializer;
    }

    public Task Execute(PlaceOrderCommand command, CancellationToken ct) =>
        _transaction.Execute(async innerCt =>
        {
            var order = await _orders.GetById(command.OrderId, innerCt)
                ?? throw new ApplicationException("Order was not found.");

            var updated = order.Place(_clock.UtcNow);

            await _orders.Save(updated.ClearPendingDomainEvents(), innerCt);

            foreach (var domainEvent in updated.PendingDomainEvents)
            {
                var integrationEvent = Map(domainEvent);
                if (integrationEvent is null)
                    continue;

                var outboxMessage = new OutboxMessage(
                    Guid.NewGuid(),
                    integrationEvent.GetType().Name,
                    _serializer.Serialize(integrationEvent),
                    integrationEvent.OccurredOnUtc,
                    null,
                    null);

                await _outbox.Add(outboxMessage, innerCt);
            }
        }, ct);

    private static IIntegrationEvent? Map(IDomainEvent domainEvent) =>
        domainEvent switch
        {
            OrderPlaced e => new OrderPlacedIntegrationEvent(
                Guid.NewGuid(),
                e.OrderId.Value,
                e.CustomerId.Value,
                e.OccurredOnUtc),
            _ => null
        };
}
```

The key idea: the use case **does not send an event to the broker**. It only stores an outbox record in the same transaction.

## Infrastructure.Persistence

Place the following in `Infrastructure.Persistence`:

- repository-port implementations
- transactional wrappers
- EF Core mappings / SQL / Dapper / other storage technology
- outbox persistence in the same database and the same transaction

Requirements:

- persistence models must not dictate the shape of the domain
- mapping between storage model ↔ domain model must be explicit
- the unit of work / transaction boundary must guarantee atomicity of data + outbox

### EF Core value object mapping with OwnsOne

When using the class-based rich model approach with EF Core, map value objects via `OwnsOne` to keep the domain clean of ORM attributes:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Account>(entity =>
    {
        entity.HasKey(a => a.Id);

        entity.OwnsOne(a => a.AccountNumber, an =>
        {
            an.Property(x => x.Value)
              .HasColumnName("AccountNumber")
              .HasMaxLength(10)
              .IsRequired();
        });

        entity.OwnsOne(a => a.Balance, money =>
        {
            money.Property(m => m.Amount)
                 .HasColumnName("Balance")
                 .HasPrecision(18, 2);

            money.OwnsOne(m => m.Currency, currency =>
            {
                currency.Property(c => c.Code)
                       .HasColumnName("Currency")
                       .HasMaxLength(3);
            });
        });

        // Domain events are transient — never persist them
        entity.Ignore(a => a.DomainEvents);
    });
}
```

### EF Core interceptor for automatic domain event publishing

When using the class-based approach, domain events can be published automatically via a `SaveChangesInterceptor`. This removes event-publishing responsibility from repositories and use cases:

```csharp
public class DomainEventPublishingInterceptor : SaveChangesInterceptor
{
    private readonly IDomainEventPublisher _eventPublisher;

    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        if (eventData.Context is not null)
            await PublishDomainEventsAsync(eventData.Context, cancellationToken);

        return await base.SavingChangesAsync(eventData, result, cancellationToken);
    }

    private async Task PublishDomainEventsAsync(DbContext context, CancellationToken ct)
    {
        var aggregateRoots = context.ChangeTracker
            .Entries<AggregateRoot<Guid>>()
            .Where(e => e.Entity.DomainEvents.Any())
            .Select(e => e.Entity)
            .ToList();

        var domainEvents = aggregateRoots.SelectMany(ar => ar.DomainEvents).ToList();

        foreach (var root in aggregateRoots)
            root.ClearDomainEvents();

        foreach (var domainEvent in domainEvents)
            await _eventPublisher.PublishAsync(domainEvent, ct);
    }
}
```

Register the interceptor via DI:

```csharp
services.AddDbContext<AccountDbContext>((sp, options) =>
{
    var interceptor = sp.GetRequiredService<DomainEventPublishingInterceptor>();
    options.UseSqlServer(connectionString).AddInterceptors(interceptor);
});
```

**Important distinction:** The interceptor pattern handles **domain events** (in-process, within the bounded context). For **integration events** (cross-service), continue using the Transactional Outbox. These two mechanisms complement each other:

- Domain events → interceptor publishes before/during save → triggers in-context side effects
- Integration events → outbox records written in the same transaction → separate worker publishes to broker

## Infrastructure.Messaging

Place the following in `Infrastructure.Messaging`:

- publisher adapter
- outbox dispatcher / background worker
- retry policy, idempotency, telemetry

Algorithm:

1. take a batch of unprocessed outbox messages
2. deserialize the integration event
3. send it to the broker
4. mark the message as `ProcessedOnUtc`
5. on error, record the error and leave the message for retry

## API

The `API` layer should:

- accept the HTTP request
- validate the transport-level contract
- invoke a single use case from `Application`
- return the result

Controllers/endpoints **must not contain domain logic**.

# How to Respond When Implementing a Task

When you are asked to implement a feature with this skill, respond in this order:

1. **Ubiquitous Language** — key terms
2. **Bounded Context** — where the task lives
3. **Aggregate Design** — aggregate root, entities, value objects, invariants
4. **Events** — domain events and integration events
5. **Application Flow** — use-case sequence
6. **Project/File Structure** — which files to create and in which assemblies
7. **Code** — complete implementation
8. **Outbox Notes** — how reliable publication is ensured
9. **Tests** — what to cover with unit/integration tests

If the user asks for code only, still follow this structure silently in your reasoning and produce code that matches it.

# Code Quality Rules

Always aim for the following:

- `CancellationToken` on all async operations
- explicit domain types (`OrderId`, `CustomerId`, `Money`) instead of loose `Guid` / `decimal` / `string`
- explicit domain errors and invariants
- small use cases with one business goal
- no leakage of infrastructure details into `Domain`
- testability without external resources
- file names, namespaces, and type names aligned with the Ubiquitous Language

# What Not to Do

Never propose these as defaults:

- “let’s use CQRS because this is DDD”
- “let’s use event sourcing because we have events”
- “let’s publish directly to Kafka/RabbitMQ after `SaveChanges`”
- “let the controller assemble the domain logic itself”
- “let’s make a mutable entity with public setters and constrain it later somehow”
- “let’s create one generic repository<T> for all entities”
- “let the API DTO also be the domain model”

# When It Is Acceptable to Deviate from the Template

It is acceptable to adapt the template if:

- the user already has an established architecture
- the specific case requires read-model optimizations
- there are technical constraints imposed by the ORM or broker
- the bounded context is very small and part of the template is objectively excessive

But even when deviating, preserve the core principles:

- clean domain
- invariants inside aggregates
- orchestration in `Application`
- repository/messaging ports in `Application`
- transactional outbox for integration events

# ASP.NET Style Guidance

If you generate code for ASP.NET:

- prefer vertical slices by use case inside `Application`, but without mandatory CQRS
- the API can be Minimal API or Controllers — choose based on the project’s existing style
- perform DI composition in the entry point
- register infrastructure adapters separately via extension methods

# Supporting Skill Files

- Result template: `templates/use-case-template.md`
- Structure and code example (immutable records): `examples/order-placement-sample.md`
- Rich model example (class-based): `examples/bank-account-rich-model-sample.md`
- Review checklist: `references/review-checklist.md`

If the task is unclear, start with the result template, then verify against the example and checklist.

For the modeling approach, choose between immutable records and class-based rich model based on the criteria in the "Alternative: Class-based rich domain model" section.
