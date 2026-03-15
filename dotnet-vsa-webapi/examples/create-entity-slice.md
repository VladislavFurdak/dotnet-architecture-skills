# Example: create entity slice

This example shows a **command slice** using:

- Minimal API
- FluentValidation
- explicit validation
- EF Core for the write model
- Result pattern
- meaningful interface at a real seam
- explicit HTTP status mapping

The entity is `Shipment`.

## Recommended file placement

```text
Features/
  Shipments/
    CreateShipment/
      Endpoint.cs
      Request.cs
      Validator.cs
      Handler.cs
      Response.cs
Domain/
  Shipments/
    Shipment.cs
    ShipmentErrors.cs
Infrastructure/
  Persistence/
    AppDbContext.cs
```

## `Domain/Shipments/Shipment.cs`

```csharp
namespace Shipments.Api.Domain.Shipments;

public sealed class Shipment
{
    private Shipment()
    {
    }

    public Guid Id { get; private set; }
    public string Number { get; private set; } = string.Empty;
    public string OrderId { get; private set; } = string.Empty;
    public string RecipientEmail { get; private set; } = string.Empty;
    public string DestinationCountryCode { get; private set; } = string.Empty;
    public DateTimeOffset CreatedAtUtc { get; private set; }

    public static Shipment Create(
        Guid id,
        string number,
        string orderId,
        string recipientEmail,
        string destinationCountryCode,
        DateTimeOffset createdAtUtc)
    {
        return new Shipment
        {
            Id = id,
            Number = number,
            OrderId = orderId,
            RecipientEmail = recipientEmail,
            DestinationCountryCode = destinationCountryCode,
            CreatedAtUtc = createdAtUtc
        };
    }
}
```

## `Domain/Shipments/ShipmentErrors.cs`

```csharp
using Shipments.Api.Shared.Results;

namespace Shipments.Api.Domain.Shipments;

public static class ShipmentErrors
{
    public static Error OrderAlreadyHasShipment(string orderId) =>
        Error.Conflict(
            code: "shipments.order_already_has_shipment",
            message: $"Order '{orderId}' already has a shipment.");

    public static Error InvalidDestination(string countryCode) =>
        Error.BusinessRule(
            code: "shipments.invalid_destination",
            message: $"Destination country '{countryCode}' is not supported.");
}
```

## `Features/Shipments/CreateShipment/Request.cs`

```csharp
namespace Shipments.Api.Features.Shipments.CreateShipment;

public sealed record CreateShipmentRequest(
    string OrderId,
    string RecipientEmail,
    string DestinationCountryCode);
```

## `Features/Shipments/CreateShipment/Response.cs`

```csharp
namespace Shipments.Api.Features.Shipments.CreateShipment;

public sealed record CreateShipmentResponse(
    Guid Id,
    string Number,
    DateTimeOffset CreatedAtUtc);
```

## `Features/Shipments/CreateShipment/Validator.cs`

```csharp
using FluentValidation;

namespace Shipments.Api.Features.Shipments.CreateShipment;

public sealed class CreateShipmentRequestValidator : AbstractValidator<CreateShipmentRequest>
{
    public CreateShipmentRequestValidator()
    {
        RuleFor(x => x.OrderId)
            .NotEmpty()
            .MaximumLength(64);

        RuleFor(x => x.RecipientEmail)
            .NotEmpty()
            .EmailAddress();

        RuleFor(x => x.DestinationCountryCode)
            .NotEmpty()
            .Length(2)
            .Matches("^[A-Z]{2}$");
    }
}
```

## `Features/Shipments/CreateShipment/Handler.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Shipments.Api.Domain.Shipments;
using Shipments.Api.Infrastructure.Persistence;
using Shipments.Api.Shared.Results;

namespace Shipments.Api.Features.Shipments.CreateShipment;

public interface ICreateShipmentHandler
{
    Task<Result<CreateShipmentResponse>> HandleAsync(
        CreateShipmentRequest request,
        CancellationToken cancellationToken);
}

public interface IShipmentNumberGenerator
{
    string Next();
}

public sealed class CreateShipmentHandler(
    AppDbContext dbContext,
    IShipmentNumberGenerator numberGenerator,
    TimeProvider timeProvider,
    ILogger<CreateShipmentHandler> logger)
    : ICreateShipmentHandler
{
    private static readonly HashSet<string> SupportedDestinations =
    [
        "DE",
        "ES",
        "FR",
        "IT",
        "NL"
    ];

    public async Task<Result<CreateShipmentResponse>> HandleAsync(
        CreateShipmentRequest request,
        CancellationToken cancellationToken)
    {
        var alreadyExists = await dbContext.Shipments
            .AnyAsync(x => x.OrderId == request.OrderId, cancellationToken);

        if (alreadyExists)
        {
            return Result.Failure<CreateShipmentResponse>(
                ShipmentErrors.OrderAlreadyHasShipment(request.OrderId));
        }

        if (!SupportedDestinations.Contains(request.DestinationCountryCode))
        {
            return Result.Failure<CreateShipmentResponse>(
                ShipmentErrors.InvalidDestination(request.DestinationCountryCode));
        }

        var shipment = Shipment.Create(
            id: Guid.NewGuid(),
            number: numberGenerator.Next(),
            orderId: request.OrderId,
            recipientEmail: request.RecipientEmail,
            destinationCountryCode: request.DestinationCountryCode,
            createdAtUtc: timeProvider.GetUtcNow());

        dbContext.Shipments.Add(shipment);
        await dbContext.SaveChangesAsync(cancellationToken);

        logger.LogInformation(
            "Created shipment {ShipmentId} for order {OrderId}",
            shipment.Id,
            shipment.OrderId);

        return Result.Success(
            new CreateShipmentResponse(
                shipment.Id,
                shipment.Number,
                shipment.CreatedAtUtc));
    }
}
```

## `Features/Shipments/CreateShipment/Endpoint.cs`

```csharp
using FluentValidation;
using Shipments.Api.Shared.Http;

namespace Shipments.Api.Features.Shipments.CreateShipment;

public static class CreateShipmentEndpoint
{
    public static IEndpointRouteBuilder MapCreateShipment(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/shipments")
            .WithTags("Shipments");

        group.MapPost("/", HandleAsync)
            .WithName("CreateShipment")
            .WithSummary("Create a shipment for an order.")
            .Produces<CreateShipmentResponse>(StatusCodes.Status201Created)
            .ProducesValidationProblem(StatusCodes.Status400BadRequest)
            .ProducesProblem(StatusCodes.Status409Conflict)
            .ProducesProblem(StatusCodes.Status422UnprocessableEntity)
            .ProducesProblem(StatusCodes.Status500InternalServerError);

        return app;
    }

    private static async Task<IResult> HandleAsync(
        CreateShipmentRequest request,
        IValidator<CreateShipmentRequest> validator,
        ICreateShipmentHandler handler,
        CancellationToken cancellationToken)
    {
        var validation = await validator.ValidateAsync(request, cancellationToken);
        if (!validation.IsValid)
        {
            return TypedResults.ValidationProblem(validation.ToDictionary());
        }

        var result = await handler.HandleAsync(request, cancellationToken);

        return result.Match(
            onSuccess: value => TypedResults.Created($"/api/shipments/{value.Id}", value),
            onFailure: error => error.ToProblemHttpResult());
    }
}
```

## Why this example is shaped this way

- The endpoint is thin.
- Validation is explicit and close to the slice.
- EF Core is used because this is a transactional write use case.
- There is no generic repository.
- The interface exists only where there is a real seam: shipment number generation.
- Expected business failures return `Result`, not exceptions.
- HTTP status mapping is explicit.
