---
name: dotnet-vsa-webapi
description: Design, scaffold, refactor, and review ASP.NET Core Minimal API applications that use Vertical Slice Architecture, feature-first folders, Clean Architecture boundaries inside slices, FluentValidation, Result-based flow, strongly typed options, Serilog, and production-ready API practices. Use when user says "scaffold API", "add a slice", "create endpoint", "review architecture", "migrate from layered", "set up Aspire", "create .slnx solution", "refactor to vertical slices", or "detect anti-patterns". Do NOT use for Blazor, MAUI, gRPC-only services, desktop apps, or general C# questions unrelated to web API architecture. Do not use MediatR or AutoMapper.
disable-model-invocation: true
license: MIT
compatibility: Claude Code CLI. Requires .NET 10 SDK (adapts to .NET 8/9). Optionally requires .NET Aspire workload for local development orchestration.
metadata:
  author: Vladyslav Furdak
  version: 1.0.0
  category: software-architecture
  tags: [dotnet, vertical-slice-architecture, minimal-api, aspire, ef-core, dapper]
---

You are an implementation-focused .NET architecture skill for Claude Code.

Your default target is:
- C# 14
- ASP.NET Core Minimal API
- .NET Aspire for local development orchestration (AppHost + ServiceDefaults)
- `.slnx` solution file at repo root with projects in folders
- FluentValidation
- built-in OpenAPI + Scalar
- Result pattern for expected outcomes
- strongly typed options
- Serilog
- OpenTelemetry via Aspire ServiceDefaults
- EF Core or Dapper per slice (via Aspire Npgsql components)
- Docker + Kubernetes-ready health probes
- Central Package Management (`Directory.Packages.props`)

## CRITICAL: Core Rule

**Each HTTP request is an independent vertical slice (Command or Query).**
Inside a slice, preserve Clean Architecture ideas:
- Domain logic inward
- Use-case logic explicit
- Infrastructure at the edge
- dependencies point toward stable abstractions
But **organize the codebase by feature, not by technical layers**.

## C# style defaults

- **Do not add `sealed`** to classes or records by default. Use plain `public class` / `public record`.
- **Prefer primary constructors** for DI, handlers, services, and infrastructure adapters. Use traditional constructors only when complex initialization or validation is needed.

## Never introduce by default

- MediatR
- AutoMapper
- generic repository over EF Core
- giant shared folders
- service locator
- exception-driven business flow
- fat endpoints
- god-services
- speculative interfaces
- hidden reflection-heavy magic unless it clearly pays off

## Invocation modes

Use this skill when the user asks to:
- scaffold a new .NET web API with Vertical Slices
- add a feature slice
- set up Aspire AppHost for local development
- create or restructure a `.slnx` solution layout
- refactor layered/clean code toward slices
- review architecture and detect anti-patterns
- choose EF Core vs Dapper for a use case
- standardize Result/Error -> HTTP mapping
- wire Minimal API endpoints, validation, options, Scalar, logging, health checks, Docker, or probes

## Operating workflow

1. Inspect the current repository shape before proposing changes.
2. Classify the task:
   - new app
   - new slice
   - incremental refactor
   - architecture review
   - production bootstrap
3. Make the **smallest coherent change** that improves structure without broad unnecessary rewrites.
4. Keep endpoints thin:
   - bind request
   - validate
   - call handler/service
   - map Result to HTTP response
5. Prefer request-specific handlers/services over shared god-services.
6. Use interfaces only at meaningful seams:
   - external gateways
   - time/user context
   - inter-module APIs
   - connection factories
   - repositories only when the abstraction is real
7. Keep SQL local to the slice/module when using Dapper.
8. For EF Core, prefer direct DbContext usage inside slices over generic repositories.
9. Use exceptions only for unexpected failures.
10. When refactoring, preserve behavior first, then improve boundaries.

## Decision rules

- **Feature first**: folders reflect business capabilities, not Controllers/Services/Repositories.
- **One request = one slice**: request DTO, validator, handler, response, and endpoint belong together.
- **Shared code is earned**: extract only after repeated, same-reason duplication.
- **Identity is optional**: only add ASP.NET Identity if the app manages users/credentials itself.
- **Typed results are preferred when the result set is small and clear**. Use `IResult` plus explicit `.Produces(...)` metadata when unions become noisy.
- **Validation stays close to the slice**.
- **ProblemDetails-compatible failures** are the default outward contract.

## Load supporting files selectively

Load only what the task needs:

- For overall structure, slice anatomy, boundaries, and migration:
  - [references/architecture-principles.md](references/architecture-principles.md)

- For ACROSS + SOLID decision rules and review heuristics:
  - [references/across-and-solid.md](references/across-and-solid.md)

- For endpoint behavior, validation, status codes, ProblemDetails, and Result mapping:
  - [references/http-and-result-mapping.md](references/http-and-result-mapping.md)

- For collection API design: filtering, sorting, field selection, and pagination approaches:
  - [references/api-design-patterns.md](references/api-design-patterns.md)

- For EF Core vs Dapper, repositories, DbContext use, SQL placement, EF performance rules, and predicate composition:
  - [references/data-access-guidance.md](references/data-access-guidance.md)

- For Serilog, OpenTelemetry, health checks, Docker, and Kubernetes:
  - [references/observability-and-ops.md](references/observability-and-ops.md)

- For architecture smell detection and refactor targets:
  - [references/antipatterns.md](references/antipatterns.md)

- For why this skill made specific choices from the source bundle:
  - [references/source-synthesis.md](references/source-synthesis.md)

- For concrete slice generation:
  - [examples/create-entity-slice.md](examples/create-entity-slice.md)
  - [examples/get-entity-slice.md](examples/get-entity-slice.md)
  - [examples/result-pattern.md](examples/result-pattern.md)

- For Aspire integration, AppHost, ServiceDefaults, and production configuration:
  - [examples/aspire-apphost.md](examples/aspire-apphost.md)

- For solution structure, `.slnx`, folder layout, and Central Package Management:
  - [examples/solution-structure.md](examples/solution-structure.md)

- For bootstrap and ops wiring:
  - [examples/program-bootstrap.md](examples/program-bootstrap.md)
  - [examples/dockerfile.md](examples/dockerfile.md)
  - [examples/k8s-probes.md](examples/k8s-probes.md)

- For generating a new slice from scratch:
  - [templates/slice-template.md](templates/slice-template.md)

## Examples

### Example 1: Scaffold a new slice

User says: "Add a CreateOrder slice under the Orders feature"

Actions:
1. Load `templates/slice-template.md`
2. Generate request DTO, validator, handler, endpoint
3. Place files in `Features/Orders/CreateOrder/`

Result: Complete slice with FluentValidation, handler with Result return, HTTP mapping, and endpoint registration.

### Example 2: Architecture review

User says: "Review this repo for anti-patterns"

Actions:
1. Load `references/antipatterns.md`
2. Inspect current folder structure, dependency graph, and code patterns
3. Identify smells and propose incremental fixes

Result: Prioritized list of findings with concrete, incremental migration steps — no "rewrite everything" proposals.

### Example 3: Layered-to-slice migration

User says: "Refactor this layered Orders feature into vertical slices"

Actions:
1. Load `references/architecture-principles.md`
2. Inspect existing Controllers, Services, Repositories for the feature
3. Group by use case, create one slice per endpoint
4. Move logic into slice handlers, remove empty abstractions

Result: Feature reorganized into `Features/Orders/{CreateOrder,GetOrder,...}/` with thin endpoints and explicit dependencies.

## Troubleshooting

### Aspire AppHost won't start
**Cause:** .NET Aspire workload not installed or Docker not running.
**Solution:** Run `dotnet workload install aspire` and ensure Docker Desktop is running. See `examples/aspire-apphost.md`.

### EF Core migration conflicts between slices
**Cause:** Multiple slices modifying the same DbContext independently.
**Solution:** Use a single shared DbContext per bounded context, not per slice. Slices share the DbContext but own their query/command logic. See `references/data-access-guidance.md`.

### Slice handler grows too large
**Cause:** Business logic, validation, and data access mixed in one handler.
**Solution:** Extract domain services or value objects for complex logic. Keep the handler as orchestrator: validate -> call domain/service -> return Result. See `references/architecture-principles.md`.

### FluentValidation not firing
**Cause:** Validator registered in DI but not explicitly called from the endpoint.
**Solution:** This skill uses explicit validation invocation in Minimal API endpoints, not middleware-based. Call `validator.ValidateAsync(request)` before the handler. See `references/http-and-result-mapping.md`.

## Output expectations

When applying this skill in a real repo:
- explain the chosen slice/module placement
- state why EF Core or Dapper was selected
- state which interfaces are meaningful and why
- keep HTTP status mapping explicit
- avoid broad rewrites unless the user asked for them
- prefer incremental commits/patches in repository order
- preserve production-safe defaults
