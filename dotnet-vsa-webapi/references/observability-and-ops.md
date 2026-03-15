# Observability and ops

This file defines the production-readiness baseline for apps built with this skill.

## Logging default: Serilog

Use Serilog as the structured application logger.

## Goals

- structured logs
- consistent correlation
- low-noise request logging
- actionable failures
- useful production diagnostics

## Baseline setup

Use:
- `Serilog.AspNetCore`
- console sink
- enrich from log context
- request logging middleware

Prefer bootstrap similar to:

```csharp
builder.Host.UseSerilog((context, services, logger) =>
{
    logger
        .ReadFrom.Configuration(context.Configuration)
        .ReadFrom.Services(services)
        .Enrich.FromLogContext()
        .WriteTo.Console();
});
```

## Log what matters

Good log events:
- startup and shutdown milestones
- dependency initialization failures
- meaningful business state changes when operationally useful
- external dependency failures
- unexpected exceptions
- key command completion events at information level when justified

Avoid:
- logging every method call
- duplicating framework noise
- logging PII/secrets
- logging the same failure in multiple layers unless each layer adds distinct operational value

## Request logging

Use request logging middleware once.

Do not also add custom duplicate “every request” logs unless they add specific business value.

Include:
- route
- status code
- elapsed time
- correlation/trace identifiers

## Correlation

Expose and log correlation identifiers:
- `TraceId`
- request id
- user id if appropriate and safe
- tenant id if multi-tenant and safe

Also include `traceId` in failure payload extensions when possible.

## OpenTelemetry: when to add it

OpenTelemetry is optional, not mandatory.

Add it when the app needs:
- distributed tracing across services
- metrics export
- log export pipelines
- standard vendor-neutral telemetry

Skip or defer it when:
- the app is small and local
- console/file logging is enough
- there is no collector/exporter story yet

## Recommended pragmatic OpenTelemetry baseline

Traces:
- ASP.NET Core instrumentation
- outgoing HTTP instrumentation
- database instrumentation if needed

Metrics:
- ASP.NET Core
- runtime
- process if your environment benefits

Logs:
- keep Serilog as app logger
- add OTel log export only when the platform requires it
- avoid dual-export complexity unless you have a clear reason

## Health checks

Expose separate endpoints for liveness and readiness.

### Liveness

Purpose:
- is the process alive?

Should usually check:
- self only

### Readiness

Purpose:
- can the instance safely receive traffic?

Should usually check:
- database availability
- required internal dependencies
- critical downstream dependencies only if the app truly cannot serve without them

Do not overload liveness with dependency checks that cause unnecessary restarts.

## Health check tags pattern

Use tags:

- `live`
- `ready`

Example:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"])
    .AddDbContextCheck<AppDbContext>(tags: ["ready"]);
```

Then map:

```csharp
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("live")
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});
```

## Docker baseline

Use a multi-stage Dockerfile.

Goals:
- deterministic build
- smaller runtime image
- explicit port
- production publish
- easy container orchestration

Baseline:
- SDK image for build/publish
- ASP.NET runtime image for final
- `ASPNETCORE_URLS=http://+:8080`
- expose 8080
- published output only in runtime image

## Kubernetes baseline

Define:
- liveness probe -> `/health/live`
- readiness probe -> `/health/ready`

Suggested defaults:
- readiness starts earlier
- liveness has more tolerance than readiness during startup
- do not make probes too aggressive for cold starts

## Graceful shutdown

ASP.NET Core handles `SIGTERM` (Kubernetes pod termination, `docker stop`, systemd) via `IHostApplicationLifetime`.
The host will by default wait for in-flight requests to finish before stopping, but you should configure explicit behavior.

### Goals

- drain in-flight requests before exit
- stop accepting new requests immediately
- flush logs and telemetry
- close database connections and background services cleanly
- respect Kubernetes `terminationGracePeriodSeconds`

### Baseline setup

```csharp
builder.Services.Configure<HostOptions>(options =>
{
    options.ShutdownTimeout = TimeSpan.FromSeconds(30);
});
```

This ensures the host waits up to 30 seconds for graceful completion before force-killing.

### Logging shutdown events

```csharp
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
```

### Kubernetes alignment

- Set `terminationGracePeriodSeconds` in the pod spec to match or exceed `ShutdownTimeout`.
- The readiness probe will fail naturally during shutdown because the host stops accepting new connections.
- Use `preStop` hook if you need extra drain time before SIGTERM.

```yaml
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: app
      lifecycle:
        preStop:
          exec:
            command: ["sh", "-c", "sleep 5"]
```

The `sleep 5` allows the Kubernetes service to remove the pod from endpoints before the app starts draining.

### Background services

If the app has `IHostedService` / `BackgroundService` implementations, they receive the cancellation signal via the `CancellationToken` passed to `ExecuteAsync` / `StopAsync`. Ensure they respect it:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        // do work
        await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
    }
}
```

### Docker

`docker stop` sends `SIGTERM` with a default 10-second grace period. Increase it if needed:

```bash
docker stop --time 30 <container>
```

Or in `docker-compose.yml`:

```yaml
services:
  app:
    stop_grace_period: 30s
```

## Configuration and secrets

- use strongly typed options
- validate critical options on startup
- keep secrets out of source code
- prefer environment variables / secret stores in deployed environments
- avoid magic strings across the codebase

## Failure handling

Expected business outcomes:
- handled via `Result`
- mapped to HTTP explicitly

Unexpected failures:
- bubble to centralized exception handling
- log once with enough context
- return generic safe 500 payload

Do not leak internal exception details in production responses.

## Local development defaults

In Development mode, redirect the root URL (`/`) to Scalar so that opening the app in a browser immediately shows the API reference with all endpoints:

```csharp
if (app.Environment.IsDevelopment())
{
    app.MapGet("/", () => Results.Redirect("/scalar/v1"))
        .ExcludeFromDescription();
}
```

`.ExcludeFromDescription()` prevents the redirect from appearing in the OpenAPI spec.

## Minimal production bootstrap checklist

- Serilog configured
- optional OTel configured only when needed
- OpenAPI + Scalar configured
- health checks mapped
- HTTPS redirection considered for deployment model
- options validated on startup
- structured exception handling in place
- Dockerfile present
- K8s probes documented
- database migrations strategy defined

## Review checklist

- Is logging structured and low-noise?
- Is request logging configured once?
- Are traces/metrics added only when justified?
- Are health checks split into live/ready?
- Are probes aligned with mapped endpoints?
- Are secrets/config strongly typed and validated?
- Is the Dockerfile production-oriented?
- Is failure handling safe and observable?
