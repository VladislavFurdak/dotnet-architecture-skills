# dotnet-vsa-webapi

> **Note:** This README is for human visitors on GitHub. When packaging this skill for upload to Claude.ai (Settings > Capabilities > Skills), exclude this file — the official skill guide requires that skill folders contain only `SKILL.md` and supporting directories (`references/`, `examples/`, `templates/`), not `README.md`.

Skill by Vladyslav Furdak.

A production-grade Claude Code skill for building, refactoring, and reviewing .NET web applications that use:

- Vertical Slice Architecture
- feature-first decomposition
- Clean Architecture boundaries inside slices
- Minimal APIs
- .NET Aspire (AppHost for local dev, ServiceDefaults for OpenTelemetry/health/resilience)
- `.slnx` solution format with Central Package Management
- FluentValidation
- Result-based flow
- strongly typed options
- Serilog
- OpenTelemetry via Aspire ServiceDefaults
- EF Core and/or Dapper (via Aspire Npgsql components)
- production-ready API and ops defaults

## Why this skill is manual-invocation only

This skill intentionally sets:

```yaml
disable-model-invocation: true
```

Reason:

- it is a **high-context architecture playbook**, not a tiny formatting preference
- it should be loaded deliberately when the session is explicitly about architecture, slice generation, or refactoring
- automatic invocation would make unrelated .NET sessions noisier and more opinionated than necessary
- this pack includes supporting references and examples meant to be loaded **selectively**, not injected by default into every ASP.NET discussion

Use it when you want Claude to actively follow this architecture and change strategy.

## Install

Project-local skill:

```bash
mkdir -p .claude/skills/dotnet-vsa-webapi
```

Then place the files from this pack into:

```text
.claude/skills/dotnet-vsa-webapi/
```

Or install it as a personal skill under:

```text
~/.claude/skills/dotnet-vsa-webapi/
```

## Invoke

Typical uses:

```text
/dotnet-vsa-webapi scaffold a new shipments API on PostgreSQL
/dotnet-vsa-webapi add a create invoice slice under Billing
/dotnet-vsa-webapi refactor this layered Orders feature into vertical slices
/dotnet-vsa-webapi review the repo for anti-patterns and propose an incremental migration
```

## What this skill optimizes for

- one HTTP request = one slice
- feature-oriented code navigation
- explicit dependencies
- low boilerplate
- Result-based business flow
- thin Minimal API endpoints
- meaningful interfaces only
- production-safe defaults
- incremental refactoring without “rewrite everything”

## What it rejects

- MediatR as a default dependency
- AutoMapper as a default dependency
- giant Application/Infrastructure/API folder trees as the primary organization model
- generic repository cargo culting
- fat controllers/endpoints
- god-services
- exception-driven business flow
- static mutable state
- shared dumping grounds
- reflection-heavy hidden magic

## Default technology opinion

For new work, the skill assumes:

- .NET 10 / C# 14
- ASP.NET Core Minimal API
- `.slnx` solution file at repo root with `Directory.Build.props` and `Directory.Packages.props`
- .NET Aspire AppHost for local development (PostgreSQL container auto-starts)
- Aspire ServiceDefaults for OpenTelemetry, health checks, resilience, and service discovery
- Aspire Npgsql components for EF Core and Dapper connection management
- built-in OpenAPI generation + Scalar UI
- FluentValidation invoked explicitly from Minimal API slices
- Serilog for structured logging
- EF Core for write-heavy aggregate work
- Dapper for targeted read/query slices
- ASP.NET Identity only when the application owns user accounts and credentials

Production connection strings come from `appsettings.json` (`ConnectionStrings:shipments-db`) or environment variables (`ConnectionStrings__shipments-db`). No Aspire orchestration needed in production.

If a repository is already on .NET 8 or .NET 9, keep the architecture rules and adapt the package/runtime details instead of forcing a framework upgrade.

## File map

- `SKILL.md`  
  Compact operational playbook for Claude Code.

- `references/architecture-principles.md`  
  Main architectural rules, slice anatomy, shared-code policy, and migration strategy.

- `references/across-and-solid.md`  
  Practical ACROSS and SOLID translation into code-generation and review heuristics.

- `references/http-and-result-mapping.md`  
  Result/Error taxonomy, HTTP mapping, validation, ProblemDetails, and endpoint conventions.

- `references/data-access-guidance.md`  
  EF Core vs Dapper decision rules, repository guidance, and slice-local SQL patterns.

- `references/observability-and-ops.md`  
  Logging, tracing, health checks, Docker, and Kubernetes defaults.

- `references/antipatterns.md`  
  Smell catalog for repo reviews and refactors.

- `references/source-synthesis.md`  
  Documents what was synthesized from the requested source bundle and how conflicts were resolved.

- `examples/aspire-apphost.md`
  Aspire AppHost, ServiceDefaults, and API integration for zero-config local development.

- `examples/solution-structure.md`
  `.slnx` solution format, folder layout, `Directory.Build.props`, and Central Package Management.

- `examples/*`
  Concrete, realistic examples for slice creation, queries, bootstrap, result mapping, Docker, and probes.

- `templates/slice-template.md`
  A reusable scaffold template for generating a new slice in a live repository.

## Recommended usage pattern in a real repository

1. Invoke the skill with the goal.
2. Let Claude inspect the current repo first.
3. Ask for an incremental plan.
4. Apply one slice/module change at a time.
5. Run tests after each coherent change.
6. Only extract shared logic when duplication is real and has the same reason to change.

## Notes on authentication

This skill does **not** force ASP.NET Identity into every sample.

Use Identity only when:
- the application owns users and passwords
- you need registration, password reset, lockout, claims/roles, etc.

If the app uses an external identity provider, prefer plain ASP.NET authentication/authorization integration and keep that concern outside unrelated slices.

## Notes on Scalar and SDK generation

This pack standardizes on Scalar for API reference UI in the web app itself.

For generated SDKs:
- keep the web app target framework and any generated client SDK target separate in your mental model
- the API app owns OpenAPI generation and documentation
- SDK generation consumes the OpenAPI document; it is not a reason to contort runtime architecture
- do not mix “app runtime choices” with “client codegen choices”

## If you extend this skill

Good additions include:
- a repo-specific `CLAUDE.md`
- additional Aspire resources (Redis, RabbitMQ, Azure services)
- organization-wide auth conventions
- organization-wide telemetry exporters
- database naming and migration rules
- architecture tests
- integration test conventions
