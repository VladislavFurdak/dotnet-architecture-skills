# Example: Dockerfile

Use a multi-stage build and a dedicated runtime image.

## `Dockerfile`

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY ["src/Shipments.Api/Shipments.Api.csproj", "src/Shipments.Api/"]
RUN dotnet restore "src/Shipments.Api/Shipments.Api.csproj"

COPY . .
WORKDIR /src/src/Shipments.Api
RUN dotnet publish "Shipments.Api.csproj" \
    -c Release \
    -o /app/publish \
    /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app

ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080

COPY --from=build /app/publish .

ENTRYPOINT ["dotnet", "Shipments.Api.dll"]
```

## Notes

- Keep the final image runtime-only.
- Use `8080` consistently with Kubernetes and platform routing.
- Prefer environment-driven configuration in deployment environments.
- Do not bake secrets into the image.
- Use readiness/liveness probes in Kubernetes instead of relying on Docker-only health assumptions.
