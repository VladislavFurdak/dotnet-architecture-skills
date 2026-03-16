# Example: .NET Aspire integration

This example shows how to integrate .NET Aspire for zero-config local development while keeping production configuration clean and environment-driven.

## Goals

- local `dotnet run` on AppHost starts PostgreSQL container automatically
- no manual database setup for new developers
- production uses connection strings from `appsettings.json` or environment variables
- ServiceDefaults project provides consistent OpenTelemetry, health checks, and resilience

## Solution structure

```text
Solution.slnx                        # root solution file
src/
  Aspire/
    AppHost/
      AppHost.csproj
      Program.cs
    ServiceDefaults/
      ServiceDefaults.csproj
      Extensions.cs
  Shipments.Api/
    Shipments.Api.csproj
    Program.cs
    ...
tests/
  Shipments.Api.Tests/
    Shipments.Api.Tests.csproj
```

## AppHost project

### `AppHost.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <IsAspireHost>true</IsAspireHost>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Aspire.Hosting.AppHost" />
    <PackageReference Include="Aspire.Hosting.PostgreSQL" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\Shipments.Api\Shipments.Api.csproj" />
  </ItemGroup>

</Project>
```

### `Program.cs`

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres")
    .WithPgAdmin()
    .WithDataVolume(isReadOnly: false)
    .WithLifetime(ContainerLifetime.Persistent);

var shipmentsDb = postgres.AddDatabase("shipments-db");

builder.AddProject<Projects.Shipments_Api>("shipments-api")
    .WithReference(shipmentsDb)
    .WaitFor(shipmentsDb)
    .WithExternalHttpEndpoints();

builder.Build().Run();
```

Key points:
- `WithDataVolume(isReadOnly: false)` persists PostgreSQL data across container recreation — survives `docker rm`
- `WithLifetime(ContainerLifetime.Persistent)` keeps the container alive across AppHost restarts — avoids re-seeding on every F5
- `WithPgAdmin()` gives you a browser-based DB UI at no cost during development
- `WaitFor(shipmentsDb)` ensures the API doesn't start until the database is accepting connections
- `WithExternalHttpEndpoints()` marks the API as externally accessible — required for Docker Compose deployment via `aspire publish`
- the resource name `"shipments-db"` becomes the connection string key automatically

## ServiceDefaults project

### `ServiceDefaults.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <IsAspireSharedProject>true</IsAspireSharedProject>
  </PropertyGroup>

  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Http.Resilience" />
    <PackageReference Include="Microsoft.Extensions.ServiceDiscovery" />
    <PackageReference Include="Npgsql.OpenTelemetry" />
    <PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" />
    <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Http" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Runtime" />
  </ItemGroup>

</Project>
```

### `Extensions.cs`

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Diagnostics.HealthChecks;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using OpenTelemetry;
using OpenTelemetry.Metrics;
using OpenTelemetry.Trace;

namespace ServiceDefaults;

public static class Extensions
{
    public static TBuilder AddServiceDefaults<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();

        builder.Services.AddServiceDiscovery();
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();
            http.AddServiceDiscovery();
        });

        return builder;
    }

    public static TBuilder ConfigureOpenTelemetry<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });

        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation()
                    .AddNpgsql();
            });

        builder.AddOpenTelemetryExporters();

        return builder;
    }

    private static TBuilder AddOpenTelemetryExporters<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        var useOtlpExporter = !string.IsNullOrWhiteSpace(
            builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"]);

        if (useOtlpExporter)
        {
            builder.Services.AddOpenTelemetry().UseOtlpExporter();
        }

        return builder;
    }

    public static TBuilder AddDefaultHealthChecks<TBuilder>(this TBuilder builder)
        where TBuilder : IHostApplicationBuilder
    {
        builder.Services.AddHealthChecks()
            .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

        return builder;
    }

    public static WebApplication MapDefaultEndpoints(this WebApplication app)
    {
        app.MapHealthChecks("/health/live", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("live")
        });

        app.MapHealthChecks("/health/ready", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("ready")
        });

        return app;
    }
}
```

## API project integration

### Additional packages for the API project

```text
Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
```

This single Aspire component replaces manual `UseNpgsql()` wiring and adds:
- connection string resolution (from Aspire orchestration or `ConnectionStrings` config)
- health checks for PostgreSQL
- OpenTelemetry tracing for Npgsql
- retry/resilience on transient failures

### `Shipments.Api.csproj` references

```xml
<ItemGroup>
  <ProjectReference Include="..\Aspire\ServiceDefaults\ServiceDefaults.csproj" />
</ItemGroup>

<ItemGroup>
  <PackageReference Include="Aspire.Npgsql.EntityFrameworkCore.PostgreSQL" />
</ItemGroup>
```

### Database registration in `Program.cs`

```csharp
// ServiceDefaults: OpenTelemetry, health checks, resilience
builder.AddServiceDefaults();

// Aspire component: resolves connection string by resource name,
// adds health check, tracing, and retry automatically
builder.AddNpgsqlDbContext<AppDbContext>("shipments-db");
```

The connection string is resolved in this order:
1. **Aspire orchestration** — when running via AppHost, injected automatically
2. **`ConnectionStrings:shipments-db`** in `appsettings.json` — standard .NET config
3. **`ConnectionStrings__shipments-db`** environment variable — production deployment

### Dapper connection factory with Aspire

When slices use Dapper alongside EF Core, register a connection factory that uses the same Aspire-managed connection string:

```csharp
builder.AddNpgsqlDataSource("shipments-db");
```

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

This way both EF Core and Dapper share the same Aspire-managed data source with health checks, tracing, and resilience.

## Production configuration

### `appsettings.json` (production)

```json
{
  "ConnectionStrings": {
    "shipments-db": "Host=db.prod.internal;Port=5432;Database=shipments;Username=app;Password=secret;SSL Mode=Require;Trust Server Certificate=false"
  }
}
```

### Environment variables (preferred for containers)

```bash
ConnectionStrings__shipments-db="Host=db.prod.internal;Port=5432;Database=shipments;Username=app;Password=secret;SSL Mode=Require"
```

### Best production practices

- **Never store secrets in `appsettings.json` committed to source control.** Use `appsettings.Production.json` excluded from git, environment variables, or a secret store.
- **Use managed identity or IAM auth** when the database supports it (e.g., Azure Managed Identity, AWS IAM, GCP IAM) to eliminate password management.
- **Use `appsettings.{Environment}.json`** layering: base `appsettings.json` has non-sensitive defaults, environment-specific files override per deployment.
- **Prefer environment variables in containerized deployments** — Kubernetes Secrets, Docker secrets, or orchestrator-injected env vars.
- **Connection pooling**: Aspire Npgsql components configure `NpgsqlDataSource` which manages pooling. Do not create `NpgsqlConnection` manually.
- **SSL/TLS**: always require encrypted connections in production (`SSL Mode=Require` or `SSL Mode=VerifyFull`).

## Running locally

```bash
# From the AppHost project — starts PostgreSQL container + API
dotnet run --project src/Aspire/AppHost

# Aspire dashboard opens at https://localhost:15xxx
# API available at https://localhost:7xxx (or http://localhost:5xxx)
# PgAdmin available at http://localhost:5050 (via WithPgAdmin)
```

New developers only need Docker running — no local PostgreSQL install, no connection string setup.

## Running in production (without Aspire orchestration)

```bash
# The API runs standalone — no AppHost needed
dotnet run --project src/Shipments.Api

# Or as a container
docker run -e ConnectionStrings__shipments-db="Host=..." shipments-api:latest
```

The Aspire components (`Aspire.Npgsql.EntityFrameworkCore.PostgreSQL`) work fine without the AppHost. They fall back to standard `ConnectionStrings` configuration.

## Docker Compose deployment via `aspire publish`

Aspire can generate production-ready Docker Compose files from the AppHost definition.

### Setup

Add the Docker hosting package to AppHost:

```xml
<PackageReference Include="Aspire.Hosting.Docker" />
```

Ensure API projects use `WithExternalHttpEndpoints()` (already shown above).

### Generate deployment artifacts

```bash
# From the AppHost project directory
aspire publish -o ../../../deploy/docker-compose
```

This produces:
- `docker-compose.yaml` — all services, databases, and networking
- `.env` — connection strings and configuration

### When to use `aspire publish`

- Deploying to a Docker-only environment (no Kubernetes)
- Quick staging/demo deployments
- Teams that prefer Docker Compose over Helm/Kustomize for simplicity

### When NOT to use `aspire publish`

- Production Kubernetes deployments — use Helm charts or Kustomize instead
- When the team already has a mature CI/CD pipeline with custom Docker Compose files
- When you need fine-grained control over container networking or resource limits

## When NOT to use Aspire

- If the team already has a working `docker-compose.yml` setup and switching has no clear benefit
- If the project targets .NET 8 without Aspire packages available
- If the deployment story is fully Kubernetes-native and the team prefers Helm/Kustomize for local dev

Aspire is opt-in infrastructure, not a requirement. The architecture (VSA, slices, Result pattern) works identically with or without it.

## Anti-patterns

- **Do not put business logic in the AppHost.** It is orchestration only.
- **Do not hardcode connection strings in `Program.cs`.** Let Aspire or configuration resolve them.
- **Do not skip `WaitFor` for database dependencies.** Race conditions on startup cause confusing errors.
- **Do not create separate `IConfiguration` sections for Aspire-managed resources.** Use `ConnectionStrings` — that is the standard the components expect.
