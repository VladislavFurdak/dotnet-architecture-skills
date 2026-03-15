# HTTP and Result mapping

This file defines the default API behavior for slices built with Minimal APIs and the Result pattern.

## Core rule

Use `Result` / `Result<T>` for expected outcomes.
Use exceptions only for:
- truly unexpected failures
- infrastructure crashes
- programmer errors
- corruption or unrecoverable states

Expected application outcomes must be mapped to explicit HTTP responses.

## Standard request flow

1. Bind request
2. Validate request
3. Execute handler
4. Map `Result` to HTTP response
5. Attach endpoint metadata

## Result model

A practical Result model should support:

- success without payload
- success with payload
- single primary error
- optional list of validation errors
- stable machine-readable error code
- human-readable message
- optional metadata for diagnostics

### Recommended error categories

- `Validation`
- `NotFound`
- `Conflict`
- `Unauthorized`
- `Forbidden`
- `BusinessRule`
- `Failure`

You can rename them, but do not collapse everything into “Error”.

## Default HTTP mapping strategy

This is the baseline mapping. Use judgment when the domain says otherwise.

| Error type | Default status |
|---|---:|
| Validation | 400 |
| NotFound | 404 |
| Conflict | 409 |
| Unauthorized | 401 |
| Forbidden | 403 |
| BusinessRule | 422 |
| Failure | 500 |

### Success defaults

| Use case | Default status |
|---|---:|
| Create with resource location | 201 |
| Create without location | 200 or 201 |
| Update with response body | 200 |
| Update without response body | 204 |
| Delete with response body | 200 |
| Delete without response body | 204 |
| Query success | 200 |

## Decision framework for 400 vs 422

Use **400** when the request is syntactically or structurally invalid:
- malformed fields
- missing required values
- shape or format errors
- validator failure on input rules

Use **422** when the request is structurally valid but violates a business rule:
- cannot cancel an already shipped order
- cannot approve an invoice in closed period
- shipment weight exceeds business limit for selected service

## ProblemDetails policy

Prefer ProblemDetails-compatible payloads for non-success responses.

Minimum useful fields:
- `title`
- `status`
- `detail`
- `type` optional
- `extensions["code"]`
- `extensions["traceId"]`

For validation problems, return field-level errors.

## Thin endpoint pattern

Preferred Minimal API shape:

```csharp
group.MapPost("/", HandleAsync);

private static async Task<IResult> HandleAsync(
    Request request,
    IValidator<Request> validator,
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
```

## DELETE endpoints and `[FromBody]`

Minimal API automatically infers body binding for POST, PUT, and PATCH methods.
**DELETE is an exception** — the framework does not infer `[FromBody]` automatically.

If a DELETE endpoint accepts a request body, you **must** annotate the parameter with `[FromBody]`.
Without it, the parameter will be `null` at runtime and cause a startup or binding error.

```csharp
// Correct — explicit [FromBody] for DELETE
group.MapDelete("/", HandleAsync);

private static async Task<IResult> HandleAsync(
    [FromBody] DeleteChatRequest request,
    IValidator<DeleteChatRequest> validator,
    IDeleteChatHandler handler,
    CancellationToken cancellationToken)
{
    // ...
}
```

```csharp
// Wrong — will fail: Minimal API does not infer body for DELETE
private static async Task<IResult> HandleAsync(
    DeleteChatRequest request, // ← not bound, null at runtime
    ...
)
```

This applies to any DELETE endpoint that reads a JSON body (e.g., soft-delete with reason, batch delete with IDs).
If the DELETE only uses route/query parameters, `[FromBody]` is not needed.

## Typed results vs `IResult`

Use typed union results when:
- the result set is small
- the endpoint is simple
- compile-time result enforcement improves clarity

Example:
```csharp
Results<Ok<Response>, NotFound> 
```

Use `IResult` plus explicit metadata when:
- the endpoint can return many domain-specific failures
- a typed union becomes noisy
- the readability loss outweighs the benefit

If you use `IResult`, add explicit metadata:
- `.Produces<T>(200)`
- `.Produces(404)`
- `.ProducesProblem(409)`
- `.ProducesValidationProblem(400)`

## Validation policy

For Minimal APIs:
- keep validators next to the slice
- inject `IValidator<TRequest>`
- invoke validation explicitly inside the endpoint or an explicit endpoint filter
- do not rely on legacy MVC auto-validation

### Preferred default

Use explicit per-endpoint validation unless a project already has a clean filter strategy.

Why:
- obvious flow
- async-friendly
- easy to debug
- no hidden magic

## Response shape rules

### Queries

Queries return projection DTOs, not tracked entities.

Good:
```csharp
ShipmentDetailsResponse
```

Bad:
```csharp
ShipmentEntity
```

### Commands

Commands return either:
- a thin created/updated response DTO
- no body when `204` is appropriate

Do not return the entire aggregate by default.

## Endpoint metadata checklist

Every slice endpoint should usually set:

- route
- tag
- name
- summary/description when useful
- success response metadata
- failure response metadata
- auth policy when required

Example:

```csharp
group.MapGet("/{id:guid}", HandleAsync)
    .WithName("GetShipmentById")
    .WithTags("Shipments")
    .Produces<ShipmentDetailsResponse>(StatusCodes.Status200OK)
    .Produces(StatusCodes.Status404NotFound)
    .ProducesProblem(StatusCodes.Status500InternalServerError);
```

## Consistent error mapping

The same domain/application error should map consistently across the app.

Bad:
- `NotFound` -> 404 in one slice
- `NotFound` -> 400 in another

Good:
- stable taxonomy
- stable HTTP mapping
- stable error codes

## Recommended error code format

Use stable, searchable codes:

```text
shipments.order_already_has_shipment
shipments.not_found
billing.invoice_already_paid
auth.forbidden
common.unexpected_failure
```

Do not expose random exception strings as the contract.

## Exception policy

Expected:
- duplicate order
- missing entity
- invalid transition
- policy violation

Unexpected:
- database unavailable
- network timeout to dependency
- serialization failure
- bug/null reference

Expected outcomes -> `Result`
Unexpected failures -> exception + centralized handling/logging

## Minimal API helper strategy

A small shared helper is acceptable for translating domain/application errors to ProblemDetails.

Acceptable:
- `Error.ToProblemHttpResult()`
- `Result.Match(...)`
- `ValidationResult.ToDictionary()`

Avoid:
- giant generic HTTP response base frameworks
- reflection-based endpoint auto-registration magic that hides behavior
- central helpers that erase domain clarity

## CancellationToken safety in handlers

Pass `CancellationToken` through to async calls — but **only when cancellation is safe**.

### When to propagate CancellationToken

- **Read-only operations** (queries, projections) — always safe to cancel.
- **Single atomic write** — a single `SaveChangesAsync` or a single SQL statement. If the client disconnects, cancelling one atomic write doesn't leave inconsistent state.

### When NOT to propagate CancellationToken

- **Multiple write operations without a transaction** — if the token fires between writes, some will have committed and others won't, leaving the system inconsistent.
- If cancellation mid-flow risks partial state, **do not pass the token** to the write calls. Let them complete even if the client has disconnected.

### Examples

```csharp
// Safe — read-only query
public async Task<Result<ShipmentDetailsResponse>> HandleAsync(
    GetShipmentByIdQuery query,
    CancellationToken cancellationToken)
{
    using var connection = await connectionFactory.OpenConnectionAsync(cancellationToken);
    var result = await connection.QuerySingleOrDefaultAsync<ShipmentDetailsResponse>(
        new CommandDefinition(Sql, new { query.Id }, cancellationToken: cancellationToken));
    // ...
}
```

```csharp
// Safe — single atomic write
public async Task<Result<CreateShipmentResponse>> HandleAsync(
    CreateShipmentRequest request,
    CancellationToken cancellationToken)
{
    // ... build entity ...
    dbContext.Shipments.Add(shipment);
    await dbContext.SaveChangesAsync(cancellationToken); // single atomic write — safe
    // ...
}
```

```csharp
// Dangerous — multiple writes without transaction, DO NOT propagate token to writes
public async Task<Result> HandleAsync(
    TransferFundsRequest request,
    CancellationToken cancellationToken)
{
    var source = await dbContext.Accounts.FindAsync([request.SourceId], cancellationToken);
    var target = await dbContext.Accounts.FindAsync([request.TargetId], cancellationToken);

    source.Debit(request.Amount);
    target.Credit(request.Amount);

    // Do NOT pass cancellationToken here — cancellation between these calls
    // would leave money debited but not credited
    await dbContext.SaveChangesAsync(CancellationToken.None);
    await auditService.LogTransferAsync(request, CancellationToken.None);
}
```

### Rule of thumb

> If cancellation mid-handler can leave the system in an inconsistent state — use `CancellationToken.None` for the write calls and let them finish.

## Review checklist

- Are status codes explicit?
- Are validation failures mapped to 400?
- Are business rule violations mapped intentionally, not accidentally?
- Are success codes appropriate to the operation?
- Does the endpoint stay thin?
- Is validation close to the slice?
- Are ProblemDetails-compatible responses used for failures?
- Are typed results used when they improve clarity?
- Are exceptions avoided for expected flow?
