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
Microsoft.AspNetCore.OpenApi
Scalar.AspNetCore
FluentValidation
FluentValidation.DependencyInjectionExtensions
Microsoft.EntityFrameworkCore
Npgsql.EntityFrameworkCore.PostgreSQL
Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore
Serilog.AspNetCore
Serilog.Sinks.Console
OpenTelemetry.Extensions.Hosting
OpenTelemetry.Instrumentation.AspNetCore
OpenTelemetry.Instrumentation.Http
OpenTelemetry.Instrumentation.Runtime
OpenTelemetry.Exporter.OpenTelemetryProtocol
```

## `Program.cs`

```csharp
using FluentValidation;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Options;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using Scalar.AspNetCore;
using Serilog;
using Shipments.Api.Features.Shipments.CreateShipment;
using Shipments.Api.Features.Shipments.GetShipmentById;
using Shipments.Api.Infrastructure.Persistence;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((context, services, logger) =>
{
    logger
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Console();
});

builder.Services.AddOpenApi();

builder.Services.AddOptions<PostgresOptions>()
    .Bind(builder.Configuration.GetSection(PostgresOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddOptions<ObservabilityOptions>()
    .Bind(builder.Configuration.GetSection(ObservabilityOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();

builder.Services.AddDbContext<AppDbContext>((services, options) =>
{
    var postgres = services.GetRequiredService<IOptions<PostgresOptions>>().Value;
    options.UseNpgsql(postgres.ConnectionString);
});

builder.Services.AddValidatorsFromAssemblyContaining<CreateShipmentRequestValidator>();

builder.Services.AddScoped<ICreateShipmentHandler, CreateShipmentHandler>();
builder.Services.AddScoped<IGetShipmentByIdHandler, GetShipmentByIdHandler>();
builder.Services.AddSingleton<IShipmentNumberGenerator, UtcShipmentNumberGenerator>();
builder.Services.AddScoped<IDbConnectionFactory, NpgsqlConnectionFactory>();
builder.Services.AddSingleton(TimeProvider.System);

builder.Services.AddHealthChecks()
    .AddCheck("self", () => Microsoft.Extensions.Diagnostics.HealthChecks.HealthCheckResult.Healthy(), tags: ["live"])
    .AddDbContextCheck<AppDbContext>(tags: ["ready"]);

var observability = builder.Configuration
    .GetSection(ObservabilityOptions.SectionName)
    .Get<ObservabilityOptions>();

if (observability?.EnableOpenTelemetry is true)
{
    builder.Services.AddOpenTelemetry()
        .ConfigureResource(resource => resource.AddService(builder.Environment.ApplicationName))
        .WithTracing(tracing =>
        {
            tracing
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation();

            if (!string.IsNullOrWhiteSpace(observability.OtlpEndpoint))
            {
                tracing.AddOtlpExporter(options => options.Endpoint = new Uri(observability.OtlpEndpoint));
            }
        })
        .WithMetrics(metrics =>
        {
            metrics
                .AddAspNetCoreInstrumentation()
                .AddRuntimeInstrumentation();

            if (!string.IsNullOrWhiteSpace(observability.OtlpEndpoint))
            {
                metrics.AddOtlpExporter(options => options.Endpoint = new Uri(observability.OtlpEndpoint));
            }
        });
}

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

app.MapHealthChecks("/health/live", new Microsoft.AspNetCore.Diagnostics.HealthChecks.HealthCheckOptions
{
    Predicate = registration => registration.Tags.Contains("live")
}).ShortCircuit();

app.MapHealthChecks("/health/ready", new Microsoft.AspNetCore.Diagnostics.HealthChecks.HealthCheckOptions
{
    Predicate = registration => registration.Tags.Contains("ready")
}).ShortCircuit();

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

public class PostgresOptions
{
    public const string SectionName = "Postgres";

    [System.ComponentModel.DataAnnotations.Required]
    public string ConnectionString { get; init; } = string.Empty;
}

public class ObservabilityOptions
{
    public const string SectionName = "Observability";

    public bool EnableOpenTelemetry { get; init; }
    public string? OtlpEndpoint { get; init; }
}

public class UtcShipmentNumberGenerator : IShipmentNumberGenerator
{
    public string Next() => $"SHP-{DateTime.UtcNow:yyyyMMddHHmmssfff}";
}
```

## `Infrastructure/Persistence/NpgsqlConnectionFactory.cs`

```csharp
using System.Data;
using Microsoft.Extensions.Options;
using Npgsql;

namespace Shipments.Api.Infrastructure.Persistence;

public class NpgsqlConnectionFactory(IOptions<PostgresOptions> options) : IDbConnectionFactory
{
    public async Task<IDbConnection> OpenConnectionAsync(CancellationToken cancellationToken)
    {
        var connection = new NpgsqlConnection(options.Value.ConnectionString);
        await connection.OpenAsync(cancellationToken);
        return connection;
    }
}
```

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

- Built-in OpenAPI + Scalar is the current default path.
- Validation is explicit and Minimal-API friendly.
- Options are strongly typed and validated on startup.
- Serilog is the baseline logger.
- OpenTelemetry is conditional, not forced.
- Health checks are split into liveness and readiness.
- Slice endpoints are registered explicitly, not by magic scanning.
