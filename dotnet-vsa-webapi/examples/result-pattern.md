# Example: Result pattern

This is a compact Result/Error implementation for Minimal API slices.

## `Shared/ErrorCodes/CommonErrorCodes.cs`

```csharp
namespace Shipments.Api.Shared.ErrorCodes;

public static class CommonErrorCodes
{
    public const string None = "common.none";
    public const string ValidationFailed = "common.validation_failed";
    public const string InternalError = "common.internal_error";
}
```

> Each bounded context / domain area gets its own `{Area}ErrorCodes` class in `Domain/{Area}/`. See `create-entity-slice.md` for example.

## `Shared/Results/Error.cs`

```csharp
namespace Shipments.Api.Shared.Results;

public enum ErrorType
{
    Validation,
    NotFound,
    Conflict,
    Unauthorized,
    Forbidden,
    BusinessRule,
    Failure
}

public record Error(
    ErrorType Type,
    string Code,
    string Message,
    IReadOnlyDictionary<string, string[]>? ValidationErrors = null)
{
    public static Error Validation(
        string code,
        string message,
        IReadOnlyDictionary<string, string[]> errors) =>
        new(ErrorType.Validation, code, message, errors);

    public static Error NotFound(string code, string message) =>
        new(ErrorType.NotFound, code, message);

    public static Error Conflict(string code, string message) =>
        new(ErrorType.Conflict, code, message);

    public static Error Unauthorized(string code, string message) =>
        new(ErrorType.Unauthorized, code, message);

    public static Error Forbidden(string code, string message) =>
        new(ErrorType.Forbidden, code, message);

    public static Error BusinessRule(string code, string message) =>
        new(ErrorType.BusinessRule, code, message);

    public static Error Failure(string code, string message) =>
        new(ErrorType.Failure, code, message);
}
```

## `Shared/Results/Result.cs`

```csharp
using Shipments.Api.Shared.ErrorCodes;

namespace Shipments.Api.Shared.Results;

public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() =>
        new(true, Error.Failure(CommonErrorCodes.None, string.Empty));

    public static Result Failure(Error error) =>
        new(false, error);

    public static Result<T> Success<T>(T value) =>
        new(value, true, Error.Failure(CommonErrorCodes.None, string.Empty));

    public static Result<T> Failure<T>(Error error) =>
        new(default, false, error);
}

public class Result<T> : Result
{
    internal Result(T? value, bool isSuccess, Error error)
        : base(isSuccess, error)
    {
        Value = value;
    }

    public T? Value { get; }

    public TValue Match<TValue>(
        Func<T, TValue> onSuccess,
        Func<Error, TValue> onFailure)
    {
        return IsSuccess
            ? onSuccess(Value!)
            : onFailure(Error);
    }
}
```

## `Shared/Http/ErrorHttpMapping.cs`

```csharp
using Shipments.Api.Shared.Results;

namespace Shipments.Api.Shared.Http;

public static class ErrorHttpMapping
{
    public static IResult ToProblemHttpResult(this Error error)
    {
        if (error.Type is ErrorType.Validation && error.ValidationErrors is not null)
        {
            return TypedResults.ValidationProblem(error.ValidationErrors);
        }

        var statusCode = error.Type switch
        {
            ErrorType.Validation => StatusCodes.Status400BadRequest,
            ErrorType.NotFound => StatusCodes.Status404NotFound,
            ErrorType.Conflict => StatusCodes.Status409Conflict,
            ErrorType.Unauthorized => StatusCodes.Status401Unauthorized,
            ErrorType.Forbidden => StatusCodes.Status403Forbidden,
            ErrorType.BusinessRule => StatusCodes.Status422UnprocessableEntity,
            _ => StatusCodes.Status500InternalServerError
        };

        return TypedResults.Problem(
            title: error.Type.ToString(),
            detail: error.Message,
            statusCode: statusCode,
            extensions: new Dictionary<string, object?>
            {
                ["code"] = error.Code
            });
    }
}
```

## `Shared/Http/FluentValidationExtensions.cs`

```csharp
using FluentValidation.Results;

namespace Shipments.Api.Shared.Http;

public static class FluentValidationExtensions
{
    public static IReadOnlyDictionary<string, string[]> ToDictionary(this ValidationResult validationResult)
    {
        return validationResult.Errors
            .GroupBy(x => x.PropertyName)
            .ToDictionary(
                group => group.Key,
                group => group.Select(x => x.ErrorMessage).Distinct().ToArray());
    }
}
```

## Usage notes

- Commands and queries return `Result<T>`.
- Validation failures should normally be handled before the handler.
- Business-rule and not-found failures come back as `Result.Failure(...)`.
- The endpoint decides the success HTTP status.
- The shared mapper decides the failure HTTP status.
- **Error code strings are always `const string` fields** — never inline string literals. Common codes go in `Shared/ErrorCodes/CommonErrorCodes.cs`. Domain-specific codes go in `Domain/{Area}/{Entity}ErrorCodes.cs` (e.g., `ShipmentErrorCodes`).
