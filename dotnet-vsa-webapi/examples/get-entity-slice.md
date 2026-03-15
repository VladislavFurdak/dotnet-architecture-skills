# Example: get entity slice

This example shows a **query slice** using:

- Minimal API
- Dapper for a read-optimized projection
- local SQL
- a meaningful connection abstraction
- typed results where the result set stays small

The entity is `Shipment`.

## Recommended file placement

```text
Features/
  Shipments/
    GetShipmentById/
      Endpoint.cs
      Query.cs
      Handler.cs
      Response.cs
Infrastructure/
  Persistence/
    IDbConnectionFactory.cs
```

## `Infrastructure/Persistence/IDbConnectionFactory.cs`

```csharp
using System.Data;

namespace Shipments.Api.Infrastructure.Persistence;

public interface IDbConnectionFactory
{
    Task<IDbConnection> OpenConnectionAsync(CancellationToken cancellationToken);
}
```

## `Features/Shipments/GetShipmentById/Query.cs`

```csharp
namespace Shipments.Api.Features.Shipments.GetShipmentById;

public sealed record GetShipmentByIdQuery(Guid Id);
```

## `Features/Shipments/GetShipmentById/Response.cs`

```csharp
namespace Shipments.Api.Features.Shipments.GetShipmentById;

public sealed record ShipmentDetailsResponse(
    Guid Id,
    string Number,
    string OrderId,
    string RecipientEmail,
    string DestinationCountryCode,
    DateTimeOffset CreatedAtUtc);
```

## `Features/Shipments/GetShipmentById/Handler.cs`

```csharp
using Dapper;
using Shipments.Api.Infrastructure.Persistence;
using Shipments.Api.Shared.Results;

namespace Shipments.Api.Features.Shipments.GetShipmentById;

public interface IGetShipmentByIdHandler
{
    Task<Result<ShipmentDetailsResponse>> HandleAsync(
        GetShipmentByIdQuery query,
        CancellationToken cancellationToken);
}

public sealed class GetShipmentByIdHandler(
    IDbConnectionFactory connectionFactory)
    : IGetShipmentByIdHandler
{
    private const string Sql = """
        select
            s.id as Id,
            s.number as Number,
            s.order_id as OrderId,
            s.recipient_email as RecipientEmail,
            s.destination_country_code as DestinationCountryCode,
            s.created_at_utc as CreatedAtUtc
        from shipments s
        where s.id = @Id
        """;

    public async Task<Result<ShipmentDetailsResponse>> HandleAsync(
        GetShipmentByIdQuery query,
        CancellationToken cancellationToken)
    {
        using var connection = await connectionFactory.OpenConnectionAsync(cancellationToken);

        var shipment = await connection.QuerySingleOrDefaultAsync<ShipmentDetailsResponse>(
            new CommandDefinition(
                Sql,
                new { query.Id },
                cancellationToken: cancellationToken));

        return shipment is null
            ? Result.Failure<ShipmentDetailsResponse>(
                Error.NotFound(
                    code: "shipments.not_found",
                    message: $"Shipment '{query.Id}' was not found."))
            : Result.Success(shipment);
    }
}
```

## `Features/Shipments/GetShipmentById/Endpoint.cs`

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Shipments.Api.Shared.Http;

namespace Shipments.Api.Features.Shipments.GetShipmentById;

public static class GetShipmentByIdEndpoint
{
    public static IEndpointRouteBuilder MapGetShipmentById(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/shipments")
            .WithTags("Shipments");

        group.MapGet("/{id:guid}", HandleAsync)
            .WithName("GetShipmentById")
            .WithSummary("Get shipment details by id.")
            .Produces<ShipmentDetailsResponse>(StatusCodes.Status200OK)
            .Produces(StatusCodes.Status404NotFound)
            .ProducesProblem(StatusCodes.Status500InternalServerError);

        return app;
    }

    private static async Task<Results<Ok<ShipmentDetailsResponse>, NotFound, ProblemHttpResult>> HandleAsync(
        Guid id,
        IGetShipmentByIdHandler handler,
        CancellationToken cancellationToken)
    {
        var result = await handler.HandleAsync(new GetShipmentByIdQuery(id), cancellationToken);

        if (result.IsSuccess)
        {
            return TypedResults.Ok(result.Value!);
        }

        return result.Error.Type switch
        {
            ErrorType.NotFound => TypedResults.NotFound(),
            _ => TypedResults.Problem(
                title: "Unexpected failure",
                detail: result.Error.Message,
                statusCode: StatusCodes.Status500InternalServerError,
                extensions: new Dictionary<string, object?>
                {
                    ["code"] = result.Error.Code
                })
        };
    }
}
```

## Why this example is shaped this way

- This is a read slice, so Dapper is a good fit.
- SQL stays close to the slice.
- The connection factory is a meaningful seam.
- The endpoint uses typed results because the result surface is small and clear.
- The query returns a projection DTO, not the entity.
