# Example: program bootstrap

This example shows a pragmatic `Program.cs` baseline for a Vertical Slice Minimal API application.

It includes:

- built-in OpenAPI
- Scalar UI
- FluentValidation registration
- strongly typed options
- EF Core
- Serilog
- optional OpenTelemetry
- health checks
- slice endpoint registration

## Package sketch

Typical packages for this baseline:

```text
Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
Microsoft.AspNetCore.OpenApi
Scalar.AspNetCore
FluentValidation
FluentValidation.DependencyInjectionExtensions
Serilog.AspNetCore
Serilog.Sinks.Console
```

The `Aspire.Npgsql.EntityFrameworkCore.PostgreSQL` component replaces manual EF Core + Npgsql wiring. It automatically adds health checks, OpenTelemetry tracing, retry resilience, and connection string resolution via `ConnectionStrings` configuration or Aspire orchestration.

OpenTelemetry packages are provided by the ServiceDefaults project and do not need to be referenced directly by the API project.

> **Without Aspire**: if the project does not use Aspire, replace with `Microsoft.EntityFrameworkCore`, `Npgsql.EntityFrameworkCore.PostgreSQL`, `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore`, and the OpenTelemetry packages directly.

## `Program.cs`

```csharp
using FluentValidation;
using Scalar.AspNetCore;
using Serilog;
using Shipments.Api.Features.Shipments.CreateShipment;
using Shipments.Api.Features.Shipments.GetShipmentById;
using Shipments.Api.Infrastructure.Persistence;

var builder = WebApplication.CreateBuilder(args);

// Aspire ServiceDefaults: OpenTelemetry, health checks, resilience, service discovery
builder.AddServiceDefaults();

builder.Host.UseSerilog((context, services, logger) =>
{
    logger
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Console();
});

builder.Services.AddOpenApi();

// Aspire Npgsql component: resolves ConnectionStrings:shipments-db from
// Aspire orchestration (local dev) or appsettings.json / env vars (production)
builder.AddNpgsqlDbContext<AppDbContext>("shipments-db");

builder.Services.AddValidatorsFromAssemblyContaining<CreateShipmentRequestValidator>();

builder.Services.AddScoped<ICreateShipmentHandler, CreateShipmentHandler>();
builder.Services.AddScoped<IGetShipmentByIdHandler, GetShipmentByIdHandler>();
builder.Services.AddSingleton<IShipmentNumberGenerator, UtcShipmentNumberGenerator>();
builder.Services.AddScoped<IDbConnectionFactory, NpgsqlConnectionFactory>();
builder.Services.AddSingleton(TimeProvider.System);

// Readiness health check — Aspire Npgsql component auto-registers a PostgreSQL health check,
// but we tag it for the /health/ready endpoint. The liveness check comes from ServiceDefaults.
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>(tags: ["ready"]);

builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

var app = builder.Build();

app.UseSerilogRequestLogging();

app.UseExceptionHandler("/error");
app.Map("/error", () => Results.Problem(
    title: "An unexpected error occurred.",
    statusCode: StatusCodes.Status500InternalServerError));

// ServiceDefaults: maps /health/live and /health/ready endpoints
app.MapDefaultEndpoints();

app.MapOpenApi();
app.MapScalarApiReference(options =>
{
    options.Title = "Shipments API";
    options.Theme = ScalarTheme.DeepSpace;
    options.DefaultHttpClient = new(ScalarTarget.CSharp, ScalarClient.HttpClient);
});

if (app.Environment.IsDevelopment())
{
    app.MapGet("/", () => Results.Redirect("/scalar/v1"))
        .ExcludeFromDescription();
}

app.MapCreateShipment();
app.MapGetShipmentById();

var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();

lifetime.ApplicationStarted.Register(() =>
    Log.Information("Application started"));

lifetime.ApplicationStopping.Register(() =>
    Log.Information("Application is shutting down..."));

lifetime.ApplicationStopped.Register(() =>
{
    Log.Information("Application stopped");
    Log.CloseAndFlush();
});

app.Run();

public class UtcShipmentNumberGenerator : IShipmentNumberGenerator
{
    public string Next() => $"SHP-{DateTime.UtcNow:yyyyMMddHHmmssfff}";
}
```

## `Infrastructure/Persistence/NpgsqlConnectionFactory.cs`

Uses `NpgsqlDataSource` injected by the Aspire component — shares the same connection pool, health checks, and tracing as EF Core.

```csharp
using System.Data;
using Npgsql;

namespace Shipments.Api.Infrastructure.Persistence;

public class NpgsqlConnectionFactory(NpgsqlDataSource dataSource) : IDbConnectionFactory
{
    public async Task<IDbConnection> OpenConnectionAsync(CancellationToken cancellationToken)
    {
        var connection = await dataSource.OpenConnectionAsync(cancellationToken);
        return connection;
    }
}
```

> **Without Aspire**: if not using Aspire, inject `IOptions<PostgresOptions>` and create `new NpgsqlConnection(options.Value.ConnectionString)` manually.

## `Infrastructure/Persistence/AppDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Shipments.Api.Domain.Shipments;

namespace Shipments.Api.Infrastructure.Persistence;

public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Shipment> Shipments => Set<Shipment>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Shipment>(entity =>
        {
            entity.ToTable("shipments");
            entity.HasKey(x => x.Id);

            entity.Property(x => x.Number).HasMaxLength(64);
            entity.Property(x => x.OrderId).HasMaxLength(64);
            entity.Property(x => x.RecipientEmail).HasMaxLength(256);
            entity.Property(x => x.DestinationCountryCode).HasMaxLength(2);

            entity.HasIndex(x => x.OrderId).IsUnique();
        });
    }
}
```

## Why this bootstrap is the default

- **Aspire ServiceDefaults** provides OpenTelemetry, health checks, resilience, and service discovery out of the box.
- **Aspire Npgsql component** handles connection string resolution, health checks, tracing, and retry — no manual wiring.
- Built-in OpenAPI + Scalar is the current default path.
- Validation is explicit and Minimal-API friendly.
- Serilog is the baseline logger.
- Health checks are split into liveness (ServiceDefaults) and readiness (DbContext check).
- Slice endpoints are registered explicitly, not by magic scanning.
- **Production**: connection string comes from `ConnectionStrings:shipments-db` in `appsettings.json` or environment variable `ConnectionStrings__shipments-db`.
- **Local dev**: Aspire AppHost injects connection strings automatically via orchestration.
