# Source synthesis

This file records how the requested source bundle was synthesized into this skill.

## Primary architectural synthesis

## 1. Vertical Slice Architecture is the primary organizing principle

From the Anton Martyniuk articles, the strongest through-line is:

- organize by feature, not technical layer
- keep all code for a use case close together
- combine that with Clean Architecture dependency direction where complexity justifies it
- accept that some applications are pure slices, while larger ones benefit from slices plus stronger domain/infrastructure/module boundaries

This skill adopts:

- **feature-first as the primary code organization**
- **Clean Architecture inside slices/modules**
- **modular monolith evolution when the system grows**

## 2. One request = one slice

The source bundle repeatedly emphasizes use-case locality.
This skill makes it explicit and central:

- each HTTP request maps to a slice
- each slice owns its endpoint, request, validator, handler, response, and local mapping

This is the anchor rule for code generation and review.

## 3. No MediatR

The source bundle includes a direct article on refactoring away from MediatR.

This skill resolves the choice decisively:

- no MediatR
- no home-grown mediator clone as a hidden replacement
- use manual handlers and direct orchestration
- use explicit decorators only if truly needed

## 4. No AutoMapper

The requested architecture philosophy rejects hidden mapping and unnecessary ceremony.
Anton’s broader guidance is also strongly pragmatic.
This skill therefore standardizes on:

- explicit local mapping
- direct projection to DTOs
- no AutoMapper by default

## 5. Clean Architecture kept where it helps

The Jason Taylor CleanArchitecture template remains a useful reference for:

- domain/application/infrastructure separation
- dependency direction
- value objects and result concepts
- composition root discipline

This skill intentionally **does not** copy its classic layer-first project sprawl as the main folder strategy.
Instead, it keeps the dependency logic while changing the primary physical organization to slices.

## 6. Result pattern for expected flow

The source bundle explicitly calls for replacing exceptions with result-based control flow.
This skill adopts:

- `Result` / `Result<T>` for expected business/application outcomes
- exceptions only for exceptional or infrastructure failures
- explicit mapping from result/error taxonomy to HTTP status codes

## 7. Options pattern as the configuration default

The options article supports:
- strong typing
- validation
- testability
- separation of concerns

This skill standardizes on:
- named options classes where helpful
- startup validation for critical settings
- no scattered magic config strings

## 8. ACROSS becomes operational, not theoretical

The ACROSS article is valuable because it centers design on change, not dogma.
This skill translates ACROSS into concrete rules for:

- where to place code
- when to introduce interfaces
- when to extract shared logic
- how to avoid rabbit-hole refactors
- how to keep contracts domain-explicit

## Conflict resolution decisions

## A. FastEndpoints vs Minimal APIs

One requested Anton article uses FastEndpoints.
The user’s target stack explicitly requires ASP.NET Core Minimal API.

Resolution:
- keep the slice locality lessons
- use Minimal APIs as the endpoint model
- do not introduce FastEndpoints into the default stack

## B. Clean Architecture project separation vs slice-first folders

Classic Clean Architecture templates often emphasize separate projects/folders for Application, Infrastructure, Presentation.

Resolution:
- retain inward dependency rules
- do not use that as the primary navigation structure for feature-heavy systems
- prefer feature folders, optionally inside modules
- keep infrastructure grouped at module/app edges where appropriate

## C. FluentValidation old MVC auto-validation vs modern Minimal APIs

Modern Minimal API reality differs from older MVC-centric examples.

Resolution:
- validators live near the slice
- validation is invoked explicitly at the endpoint boundary by default
- optional explicit endpoint filter is acceptable if already established cleanly
- no legacy magic auto-validation recommendation for new Minimal API work

## D. Swashbuckle-era docs vs current OpenAPI + Scalar approach

Modern ASP.NET Core now has built-in OpenAPI generation, and Scalar integrates cleanly on top.

Resolution:
- built-in OpenAPI generation first
- Scalar for API reference UI
- do not anchor the skill on older Swagger-first assumptions

## E. Repository guidance

The source bundle is pragmatic and also includes thought around avoiding duplication and avoiding unnecessary repository patterns.

Resolution:
- EF Core: direct DbContext in slices unless a real repository seam exists
- Dapper: use a narrow connection abstraction and keep SQL close to the slice
- never introduce generic repositories as a blanket policy

## F. Identity

The source bundle is not “Identity everywhere”.
The user also explicitly forbids that.

Resolution:
- Identity only when auth is required and the app owns users/credentials
- otherwise keep auth out of examples and out of unrelated slices

## Final skill posture

This skill is opinionated in favor of:

- vertical slices
- manual handlers
- meaningful interfaces only
- explicit HTTP behavior
- Result-based flow
- strongly typed options
- Serilog baseline
- optional OpenTelemetry
- EF Core + Dapper by slice
- incremental refactoring
- production-safe bootstrapping

This skill is intentionally against:

- layer-first folder trees as the main structure
- MediatR as default
- AutoMapper as default
- generic repositories
- hidden magic
- big-bang rewrites
