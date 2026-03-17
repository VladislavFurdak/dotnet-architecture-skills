# Example: Order Placement with Transactional Outbox

Below is an example of how the skill should reason when implementing the `Place Order` feature.

## Ubiquitous Language

- Bounded Context: `Sales`
- Aggregate Root: `Order`
- Entity: `Order`
- Value Objects: `OrderId`, `CustomerId`, `ProductId`, `Money`, `OrderItem`
- Domain Event: `OrderPlaced`
- Integration Event: `OrderPlacedIntegrationEvent`

## Key invariant rules

- An order can be changed only while it is in `Draft` status
- An order cannot be placed without items
- Currency inside `Money` must remain consistent

## Suggested file structure

```text
src/
  Sales.Domain/
    Orders/
      Order.cs
      OrderItem.cs
      OrderId.cs
      CustomerId.cs
      ProductId.cs
      Money.cs
      OrderPlaced.cs
      DomainRuleViolation.cs

  Sales.Application/
    Abstractions/
      Persistence/
        IOrderRepository.cs
        IApplicationTransaction.cs
        IOutboxWriter.cs
      Services/
        IIntegrationEventSerializer.cs
        ISystemClock.cs
    IntegrationEvents/
      IIntegrationEvent.cs
      OrderPlacedIntegrationEvent.cs
    Outbox/
      OutboxMessage.cs
    UseCases/
      PlaceOrder/
        PlaceOrderCommand.cs
        PlaceOrderUseCase.cs

  Sales.Infrastructure.Persistence/
    Orders/
      EfOrderRepository.cs
    Outbox/
      EfOutboxWriter.cs
    Transactions/
      EfApplicationTransaction.cs

  Sales.Infrastructure.Messaging/
    Dispatching/
      OutboxDispatcherBackgroundService.cs
    Publishing/
      BrokerIntegrationEventPublisher.cs

  Sales.API/
    Endpoints/
      OrdersEndpoints.cs
```

## Shape of the use case

```csharp
public sealed record PlaceOrderCommand(OrderId OrderId);

public sealed class PlaceOrderUseCase
{
    public async Task Execute(PlaceOrderCommand command, CancellationToken ct)
    {
        await _transaction.Execute(async innerCt =>
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

                await _outbox.Add(ToOutboxMessage(integrationEvent), innerCt);
            }
        }, ct);
    }
}
```

## What not to do

### Wrong

```csharp
await _dbContext.SaveChangesAsync(ct);
await _broker.PublishAsync(new OrderPlacedIntegrationEvent(...), ct);
```

Problem: if the process crashes after `SaveChangesAsync`, the business change is already committed, but the event is lost.

### Right

```csharp
await _transaction.Execute(async innerCt =>
{
    await _orders.Save(order, innerCt);
    await _outbox.Add(outboxMessage, innerCt);
}, ct);
```

Broker publication is delegated to a separate worker.

## Review criteria

- The domain does not know about the database or broker
- The use case orchestrates, but does not contain order invariants
- The integration event is not published directly
- The outbox is written in the same transaction as the aggregate
- Names stay consistent around the term `Order` and are not mixed with `Purchase`
