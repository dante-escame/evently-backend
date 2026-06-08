# Extracting Modules to Microservices

> **Physical Boundary:** A deployment boundary where a module runs in its own process, with its own host, its own Docker container, and its own independent lifecycle.

-> Elevating a module's logical boundary to a physical one without changing its internal structure.

---

## When to Extract

A modular monolith already enforces strict logical separation between modules. Each module has its own database schema, its own assembly, and no direct references to other modules' internals.

Extraction promotes that separation to a physical level. The module gets independent deployability, its own scaling surface, and organizational autonomy. The internal code does not change. Only the host, infrastructure wiring, and cross-module communication change.

-> This guide does not cover physical database separation. If needed, manually migrate the module's schema into a separate database after extraction.

---

## Microservice Creation

Create a new ASP.NET Core Web API project with OpenAPI and Docker support. Place it under `src/API/Evently.<Context>.Api/`.

Add a `<ProjectReference>` from the new API to the module's Infrastructure project.

`src/API/Evently.Ticketing.Api/Evently.Ticketing.Api.csproj`

```xml
<ItemGroup>
  <ProjectReference Include="..\..\Modules\Ticketing\Evently.Modules.Ticketing.Infrastructure\Evently.Modules.Ticketing.Infrastructure.csproj" />
</ItemGroup>
```

Sync `<PackageReference>` entries with the original API. Both projects use the same base packages: `Serilog.AspNetCore`, `Serilog.Sinks.Seq`, `Swashbuckle.AspNetCore`, `AspNetCore.HealthChecks.UI.Client`, and `Microsoft.EntityFrameworkCore.Tools`.

Copy `appsettings.json` and `appsettings.Development.json` from the original API. Remove connection strings and settings that belong to other modules. Update the `"Application"` section to reflect the new service name.

Include the module-specific configuration file by calling `AddModuleConfiguration` with only the extracted module's key.

`src/API/Evently.Ticketing.Api/Program.cs`

```csharp
builder.Configuration.AddModuleConfiguration(["ticketing"]);
```

Copy `Program.cs`. Remove the assembly references, module registrations (`AddXModule`), and consumer configurations for every other module. Register only the extracted module's application assembly, infrastructure, and consumers.

```csharp
Assembly[] moduleApplicationAssemblies = [Evently.Modules.Ticketing.Application.AssemblyReference.Assembly];

builder.Services.AddApplication(moduleApplicationAssemblies);

builder.Services.AddInfrastructure(
    DiagnosticsConfig.ServiceName,
    [
        TicketingModule.ConfigureConsumers
    ],
    rabbitMqSettings,
    databaseConnectionString,
    redisConnectionString);

builder.Services.AddTicketingModule(builder.Configuration);
```

Create `OpenTelemetry/DiagnosticsConfig.cs` in the new API project and set `ServiceName` to the new service name.

`src/API/Evently.Ticketing.Api/OpenTelemetry/DiagnosticsConfig.cs`

```csharp
public static class DiagnosticsConfig
{
    public const string ServiceName = "Evently.Ticketing.Api";
}
```

Update all namespaces in the new project to match the new assembly (e.g., `Evently.Ticketing.Api` instead of `Evently.Api`).

In the original API, remove the extracted module:

- Delete the `<ProjectReference>` to the module's Infrastructure project from `Evently.Api.csproj`.
- Remove the module's application assembly from `moduleApplicationAssemblies`.
- Remove the module's `ConfigureConsumers` from the `AddInfrastructure` call.
- Remove the `AddXModule` call and its configuration key from `AddModuleConfiguration`.
- Remove the `using` directive for the module's Infrastructure namespace.
- Remove the module's `DbContext` from `MigrationExtensions.ApplyMigrations`.

---

## Docker

Add a `Dockerfile` to the new API project. Structure it identically to the original API Dockerfile, but include only the projects that the new API actually references.

`src/API/Evently.Ticketing.Api/Dockerfile`

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["Directory.Build.props", "."]
COPY ["src/API/Evently.Ticketing.Api/Evently.Ticketing.Api.csproj", "src/API/Evently.Ticketing.Api/"]
COPY ["src/Modules/Ticketing/Evently.Modules.Ticketing.Infrastructure/Evently.Modules.Ticketing.Infrastructure.csproj", "src/Modules/Ticketing/Evently.Modules.Ticketing.Infrastructure/"]
# ... all transitive dependency .csproj files ...
RUN dotnet restore "./src/API/Evently.Ticketing.Api/Evently.Ticketing.Api.csproj"
COPY . .
WORKDIR "/src/src/API/Evently.Ticketing.Api"
RUN dotnet build "./Evently.Ticketing.Api.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./Evently.Ticketing.Api.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "Evently.Ticketing.Api.dll"]
```

-> **Attention:** If the solution stays in a single repository, the `COPY . .` step will include the integration test projects of other modules in the build context. This does not cause build failures as long as those projects are not referenced by the new API. If the microservice is moved to a separate solution, the orphaned references in those test projects must be resolved first (see the Integration Tests section).

Also update the original API's Dockerfile to remove the `COPY` lines for the extracted module's projects.

Add the new service to `docker-compose.yml`.

`docker-compose.yml`

```yaml
evently.ticketing.api:
  image: ${DOCKER_REGISTRY-}eventlyticketingapi
  container_name: Evently.Ticketing.Api
  build:
    context: .
    dockerfile: src/API/Evently.Ticketing.Api/Dockerfile
  ports:
    - 5100:8080
    - 5101:8081
```

Add the corresponding entry in `docker-compose.override.yml` with `ASPNETCORE_ENVIRONMENT`, port bindings, and volume mounts for user secrets and HTTPS certificates.

In the new API's `MigrationExtensions`, migrate only the extracted module's `DbContext`.

`src/API/Evently.Ticketing.Api/Extensions/MigrationExtensions.cs`

```csharp
public static void ApplyMigrations(this IApplicationBuilder app)
{
    using IServiceScope scope = app.ApplicationServices.CreateScope();
    ApplyMigration<TicketingDbContext>(scope);
}
```

Update `launchSettings.json` to include the new service in the multi-service startup profile.

---

## Integration Tests

The extracted module's integration test project references the host it boots. Update the `<ProjectReference>` in `Evently.Modules.Ticketing.IntegrationTests.csproj` to point to the new API project.

`src/Modules/Ticketing/Evently.Modules.Ticketing.IntegrationTests/Evently.Modules.Ticketing.IntegrationTests.csproj`

```xml
<ProjectReference Include="..\..\..\API\Evently.Ticketing.Api\Evently.Ticketing.Api.csproj" />
```

If the existing `IntegrationTestWebAppFactory` boots the original API, create a new factory that boots the new API instead, and rename the old factory to make its scope clear (e.g., `MainApiIntegrationTestWebAppFactory`).

In the original shared integration test project (`Evently.IntegrationTests`), remove all test cases and helper methods that depended on the extracted module. Any test case that issued commands or queries into the extracted module through `ISender` no longer compiles once the module's application assembly is gone from that host.

In the architecture tests project, remove the boundary test for the extracted module. Once the module runs in its own process, the cross-module dependency check is no longer meaningful in the monolith's test suite.

---

## Data Dependencies

-> **Attention:** Watch for data that the extracted microservice needs but that is owned by a module still in the original host. This is the most common source of breakage after extraction.

When a module runs in-process, it can share data through in-memory events and projections. Once it moves to a separate process, those channels break.

The concrete example here is **authorization**. The Ticketing module's `IPermissionService` resolves user permissions on every authenticated request. In the monolith, the Users module's `PermissionService` ran in the same process. After extraction, Ticketing cannot call into Users directly.

The solution is a **MassTransit `IRequestClient`**. The Users module exposes a `GetUserPermissionsRequestConsumer` in its Presentation layer. Ticketing injects `IRequestClient<GetUserPermissionsRequest>` and sends the request over RabbitMQ. The response arrives asynchronously and Ticketing returns a typed result.

`src/Modules/Ticketing/Evently.Modules.Ticketing.Infrastructure/Authorization/PermissionService.cs`

```csharp
internal sealed class PermissionService(
    IRequestClient<GetUserPermissionsRequest> requestClient,
    ICacheService cacheService) : IPermissionService
{
    private static readonly TimeSpan CacheExpiration = TimeSpan.FromMinutes(5);

    public async Task<Result<PermissionsResponse>> GetUserPermissionsAsync(string identityId)
    {
        PermissionsResponse? cached = await cacheService.GetAsync<PermissionsResponse>(CreateCacheKey(identityId));
        if (cached is not null)
        {
            return cached;
        }

        Response<PermissionsResponse, Error> response =
            await requestClient.GetResponse<PermissionsResponse, Error>(new GetUserPermissionsRequest(identityId));

        if (response.Is(out Response<Error> errorResponse))
        {
            return Result.Failure<PermissionsResponse>(errorResponse.Message);
        }

        if (response.Is(out Response<PermissionsResponse> permissionResponse))
        {
            await cacheService.SetAsync(CreateCacheKey(identityId), permissionResponse.Message, CacheExpiration);
            return permissionResponse.Message;
        }

        return Result.Failure<PermissionsResponse>(Error.NotFound(nameof(PermissionService), "The user was not found"));
    }

    private static string CreateCacheKey(string identityId) => $"user-permissions:{identityId}";
}
```

Register `IPermissionService` in the module's DI wiring.

`src/Modules/Ticketing/Evently.Modules.Ticketing.Infrastructure/TicketingModule.cs`

```csharp
services.AddScoped<IPermissionService, PermissionService>();
```

Add the provider's `ConfigureConsumers` to the original API's `AddInfrastructure` call so the Users consumer is active and reachable on the bus.

`src/API/Evently.Api/Program.cs`

```csharp
builder.Services.AddInfrastructure(
    DiagnosticsConfig.ServiceName,
    [
        EventsModule.ConfigureConsumers(redisConnectionString),
        AttendanceModule.ConfigureConsumers,
        UsersModule.ConfigureConsumers
    ],
    ...);
```

Cache the response in Redis. Permission checks run on every authenticated request. Without caching, every request results in a RabbitMQ round-trip.

See `docs/request-client-mass-transit.md` for the full pattern.
