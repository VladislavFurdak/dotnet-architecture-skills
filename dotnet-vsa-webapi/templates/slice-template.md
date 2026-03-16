# Slice template

Use this template when adding a new request slice to an existing ASP.NET Core Minimal API application.

Replace the placeholders and keep the structure feature-first.

---

## 1. Choose the slice type

- Command: changes state
- Query: reads state

Set:

- Feature: `<FeatureName>`
- Slice: `<UseCaseName>`
- Route: `<Route>`
- Method: `<GET|POST|PUT|PATCH|DELETE>`
- Data access: `<EF Core|Dapper|Both>`
- Auth required: `<Yes|No>`
- Success status: `<200|201|204>`
- Expected failures: `<400|404|409|422|...>`

---

## 2. Recommended file layout

```text
Features/
  <FeatureName>/
    <UseCaseName>/
      Endpoint.cs
      Request.cs            # or Query.cs / Command.cs
      Validator.cs          # omit only if truly unnecessary
      Handler.cs
      Response.cs           # omit if 204
```

Optional nearby support:

```text
Features/
  <FeatureName>/
    Shared/
      ...
```

Only add `Shared/` when the logic is reused by related slices for the same reason.

---

## 3. Request model

```csharp
namespace <RootNamespace>.Features.<FeatureName>.<UseCaseName>;

public record <RequestName>(
    /* immutable request fields */
);
```

Rules:
- prefer records for request/response DTOs
- keep the request shaped for the use case, not the entity
- do not expose domain entities as request contracts

---

## 4. Validator

```csharp
using FluentValidation;

namespace <RootNamespace>.Features.<FeatureName>.<UseCaseName>;

public class <RequestName>Validator : AbstractValidator<<RequestName>>
{
    public <RequestName>Validator()
    {
        // request-shape and simple business-input validation
    }
}
```

Rules:
- keep validation close to the slice
- validate format and request-shape constraints here
- leave deeper business workflow decisions to the handler/domain

---

## 5. Response model

```csharp
namespace <RootNamespace>.Features.<FeatureName>.<UseCaseName>;

public record <ResponseName>(
    /* projection fields only */
);
```

Rules:
- return DTOs or projections
- do not leak EF entities
- for `204`, omit the response type

---

## 6. Handler contract

Only introduce an interface if the handler is consumed through a real seam.
For most app-internal handlers, the concrete class is enough.
If your codebase standardizes handler interfaces, keep them narrow and request-specific.

```csharp
namespace <RootNamespace>.Features.<FeatureName>.<UseCaseName>;

public interface I<UseCaseName>Handler
{
    Task<Result<<ResponseName>>> HandleAsync(
        <RequestName> request,
        CancellationToken cancellationToken);
}
```

---

## 7. Handler implementation

### EF Core command pattern

```csharp
public class <UseCaseName>Handler(
    AppDbContext dbContext,
    /* meaningful collaborators only */,
    ILogger<<UseCaseName>Handler> logger)
    : I<UseCaseName>Handler
{
    public async Task<Result<<ResponseName>>> HandleAsync(
        <RequestName> request,
        CancellationToken cancellationToken)
    {
        // 1. Query only what this use case needs
        // 2. Enforce business rules
        // 3. Create/update domain model
        // 4. Save changes
        // 5. Return success DTO
        // 6. Return Result.Failure(...) for expected outcomes
    }
}
```

### Dapper query pattern

```csharp
public class <UseCaseName>Handler(
    IDbConnectionFactory connectionFactory)
    : I<UseCaseName>Handler
{
    private const string Sql = """
        select ...
        """;

    public async Task<Result<<ResponseName>>> HandleAsync(
        <RequestName> request,
        CancellationToken cancellationToken)
    {
        using var connection = await connectionFactory.OpenConnectionAsync(cancellationToken);

        var response = await connection.QuerySingleOrDefaultAsync<<ResponseName>>(
            new CommandDefinition(Sql, new { /* parameters */ }, cancellationToken: cancellationToken));

        return response is null
            ? Result.Failure<<ResponseName>>(
                <Entity>Errors.NotFound(request.Id))
            : Result.Success(response);
    }
}
```

Rules:
- commands usually favor EF Core
- queries often favor Dapper when projection-heavy
- keep SQL local
- avoid generic repositories
- introduce interfaces only where the boundary is meaningful

---

## 8. Endpoint

### Explicit validation + Result mapping

```csharp
using FluentValidation;

namespace <RootNamespace>.Features.<FeatureName>.<UseCaseName>;

public static class <UseCaseName>Endpoint
{
    public static IEndpointRouteBuilder Map<UseCaseName>(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("<RoutePrefix>")
            .WithTags("<FeatureName>");

        group.Map<Verb>("<RouteSuffix>", HandleAsync)
            .WithName("<UseCaseName>")
            .WithSummary("<Business summary>")
            .Produces<<ResponseName>>(<SuccessStatusCode>)
            .ProducesValidationProblem(StatusCodes.Status400BadRequest)
            .ProducesProblem(StatusCodes.Status404NotFound)
            .ProducesProblem(StatusCodes.Status409Conflict)
            .ProducesProblem(StatusCodes.Status422UnprocessableEntity)
            .ProducesProblem(StatusCodes.Status500InternalServerError);

        return app;
    }

    // NOTE: For DELETE endpoints, annotate body parameters with [FromBody].
    // Minimal API does not infer body binding for DELETE automatically.
    private static async Task<IResult> HandleAsync(
        <BoundRequestParameters>,  // use [FromBody] if DELETE + body
        IValidator<<RequestName>> validator,
        I<UseCaseName>Handler handler,
        CancellationToken cancellationToken)
    {
        var request = <BuildOrBindRequest>();

        var validation = await validator.ValidateAsync(request, cancellationToken);
        if (!validation.IsValid)
        {
            return TypedResults.ValidationProblem(validation.ToDictionary());
        }

        var result = await handler.HandleAsync(request, cancellationToken);

        return result.Match(
            onSuccess: value => <SuccessResult>,
            onFailure: error => error.ToProblemHttpResult());
    }
}
```

Rules:
- endpoint stays thin
- success mapping is endpoint-specific
- failure mapping is standardized
- use typed union results only when it improves readability

---

## 9. Error codes and domain errors

Each domain area defines:
- `Domain/<Area>/<Entity>ErrorCodes.cs` — `const string` fields for error code keys
- `Domain/<Area>/<Entity>Errors.cs` — factory methods that return `Error` instances using those constants

```csharp
// Domain/<FeatureName>/<Entity>ErrorCodes.cs
public static class <Entity>ErrorCodes
{
    public const string NotFound = "<feature>.<entity>_not_found";
    // add more as slices grow
}

// Domain/<FeatureName>/<Entity>Errors.cs
public static class <Entity>Errors
{
    public static Error NotFound(Guid id) =>
        Error.NotFound(
            code: <Entity>ErrorCodes.NotFound,
            message: $"<Entity> '{id}' was not found.");
}
```

Never use inline string literals for error codes — always reference `*ErrorCodes` constants.

---

## 10. Domain integration

If the slice needs domain behavior, keep it explicit.

Use domain entities/value objects when they improve clarity:
- invariants
- transitions
- calculation logic
- policies tied to business language

Do not invent a domain layer for trivial CRUD without value.

---

## 11. Registration checklist

When wiring the slice into the app:

- register validator
- register handler
- register meaningful collaborators
- map endpoint in `Program.cs` or a feature registration extension

Example:

```csharp
builder.Services.AddScoped<I<UseCaseName>Handler, <UseCaseName>Handler>();
app.Map<UseCaseName>();
```

---

## 12. Review checklist before finishing

- Is the slice placed under the correct feature?
- Is one request implemented in one slice?
- Is validation close to the slice?
- Is the endpoint thin?
- Are success and failure status codes explicit?
- Is Result used instead of exceptions for expected outcomes?
- Are interfaces meaningful?
- Is EF Core or Dapper justified?
- Is shared code local unless truly earned?
- Are error code strings defined as `const` in `*ErrorCodes` classes (no inline literals)?
- Is auth added only if needed?
