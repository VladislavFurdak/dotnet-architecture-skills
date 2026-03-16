# Example: solution structure and .slnx

This example defines the recommended solution layout with a `.slnx` file at the root and projects organized into folders.

## Why `.slnx` over `.sln`

- `.slnx` is the modern XML-based solution format introduced in .NET 9+
- cleaner, more readable, easier to merge in version control
- no GUIDs for solution folders
- supported by Visual Studio 2022 17.10+, Rider 2024.2+, and `dotnet` CLI

## Root solution file

The `.slnx` file lives at the **repository root**, next to `src/` and `tests/`.

### `Solution.slnx`

```xml
<Solution>
  <Folder Name="/src/Aspire/">
    <Project Path="src/Aspire/AppHost/AppHost.csproj" />
    <Project Path="src/Aspire/ServiceDefaults/ServiceDefaults.csproj" />
  </Folder>
  <Folder Name="/src/">
    <Project Path="src/Shipments.Api/Shipments.Api.csproj" />
  </Folder>
  <Folder Name="/tests/">
    <Project Path="tests/Shipments.Api.Tests/Shipments.Api.Tests.csproj" />
  </Folder>
</Solution>
```

## Physical folder layout

```text
repo-root/
├── Solution.slnx
├── Directory.Build.props          # shared build properties
├── Directory.Packages.props       # central package management
├── .gitignore
├── src/
│   ├── Aspire/
│   │   ├── AppHost/
│   │   │   ├── AppHost.csproj
│   │   │   ├── Program.cs
│   │   │   └── Properties/
│   │   │       └── launchSettings.json
│   │   └── ServiceDefaults/
│   │       ├── ServiceDefaults.csproj
│   │       └── Extensions.cs
│   └── Shipments.Api/
│       ├── Shipments.Api.csproj
│       ├── Program.cs
│       ├── Features/
│       │   └── Shipments/
│       │       ├── CreateShipment/
│       │       └── GetShipmentById/
│       ├── Domain/
│       │   └── Shipments/
│       └── Infrastructure/
│           └── Persistence/
│               └── Configurations/
└── tests/
    └── Shipments.Api.Tests/
        └── Shipments.Api.Tests.csproj
```

## Multi-service solution

When the solution grows to multiple services, each gets its own folder under `src/`:

```xml
<Solution>
  <Folder Name="/src/Aspire/">
    <Project Path="src/Aspire/AppHost/AppHost.csproj" />
    <Project Path="src/Aspire/ServiceDefaults/ServiceDefaults.csproj" />
  </Folder>
  <Folder Name="/src/Shipments/">
    <Project Path="src/Shipments/Shipments.Api/Shipments.Api.csproj" />
  </Folder>
  <Folder Name="/src/Billing/">
    <Project Path="src/Billing/Billing.Api/Billing.Api.csproj" />
  </Folder>
  <Folder Name="/src/Shared/">
    <Project Path="src/Shared/Contracts/Contracts.csproj" />
  </Folder>
  <Folder Name="/tests/">
    <Project Path="tests/Shipments.Api.Tests/Shipments.Api.Tests.csproj" />
    <Project Path="tests/Billing.Api.Tests/Billing.Api.Tests.csproj" />
  </Folder>
</Solution>
```

Physical layout:

```text
repo-root/
├── Solution.slnx
├── src/
│   ├── Aspire/
│   │   ├── AppHost/
│   │   └── ServiceDefaults/
│   ├── Shipments/
│   │   └── Shipments.Api/
│   ├── Billing/
│   │   └── Billing.Api/
│   └── Shared/
│       └── Contracts/
└── tests/
    ├── Shipments.Api.Tests/
    └── Billing.Api.Tests/
```

## `Directory.Build.props`

Place at the repo root to share build settings across all projects:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
</Project>
```

## `Directory.Packages.props`

Central Package Management keeps package versions consistent:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageFloatingVersionsEnabled>true</CentralPackageFloatingVersionsEnabled>
  </PropertyGroup>

  <ItemGroup>
    <!-- Aspire -->
    <PackageVersion Include="Aspire.Hosting.AppHost" Version="9.*" />
    <PackageVersion Include="Aspire.Hosting.PostgreSQL" Version="9.*" />

    <!-- EF Core + Npgsql (direct — not via Aspire component, see aspire-apphost.md) -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.*" />
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.*" />
    <PackageVersion Include="Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore" Version="10.*" />

    <!-- Validation -->
    <PackageVersion Include="FluentValidation" Version="11.*" />
    <PackageVersion Include="FluentValidation.DependencyInjectionExtensions" Version="11.*" />

    <!-- Logging -->
    <PackageVersion Include="Serilog.AspNetCore" Version="9.*" />
    <PackageVersion Include="Serilog.Sinks.Console" Version="6.*" />

    <!-- API -->
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.*" />
    <PackageVersion Include="Scalar.AspNetCore" Version="2.*" />

    <!-- OpenTelemetry (via ServiceDefaults) -->
    <PackageVersion Include="Microsoft.Extensions.Http.Resilience" Version="9.*" />
    <PackageVersion Include="Microsoft.Extensions.ServiceDiscovery" Version="9.*" />
    <PackageVersion Include="OpenTelemetry.Extensions.Hosting" Version="1.*" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.*" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.Http" Version="1.*" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.Runtime" Version="1.*" />
    <PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.*" />
  </ItemGroup>
</Project>
```

Then in `.csproj` files, reference packages without versions:

```xml
<PackageReference Include="FluentValidation" />
```

## Creating the `.slnx` from scratch

```bash
# Create a new slnx solution
dotnet new sln --name Solution --format slnx

# Add projects
dotnet sln Solution.slnx add src/Shipments.Api/Shipments.Api.csproj --solution-folder src
dotnet sln Solution.slnx add src/Aspire/AppHost/AppHost.csproj --solution-folder src/Aspire
dotnet sln Solution.slnx add src/Aspire/ServiceDefaults/ServiceDefaults.csproj --solution-folder src/Aspire
dotnet sln Solution.slnx add tests/Shipments.Api.Tests/Shipments.Api.Tests.csproj --solution-folder tests
```

## Rules

- **One `.slnx` at the repo root.** Do not nest solution files in subfolders.
- **Solution folders mirror physical folders** — `/src/`, `/tests/`, `/src/Aspire/` map directly to disk.
- **Aspire projects go under `src/Aspire/`** — keeps orchestration separate from business services.
- **Each service gets its own physical folder under `src/`** — for single-service solutions, directly as `src/ServiceName/`; for multi-service, group by domain `src/Shipments/`, `src/Billing/`.
- **Tests mirror source** — `tests/Shipments.Api.Tests/` tests `src/Shipments.Api/`.
- **Use Central Package Management** — one `Directory.Packages.props` at the root.
- **Use `Directory.Build.props`** for shared compiler settings.
