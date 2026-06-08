# Distributed Tracing

> **Distributed Tracing:** A method of tracking requests as they propagate through services and components by attaching a unique trace identifier to each operation.

-> Correlating logs, spans, and errors across all modules and infrastructure layers under a single trace.

---

## What Is OpenTelemetry

**OpenTelemetry** is an open-source observability framework that standardizes how applications emit telemetry: traces, metrics, and logs. It provides vendor-neutral SDKs and a wire protocol (OTLP) for exporting data to any compatible backend.

**Jaeger** is an open-source distributed tracing backend originally built by Uber. It collects traces exported via OTLP and provides a UI for searching, filtering, and visualizing trace timelines. In Evently it runs as a Docker container and receives spans from the API process.

---

## NuGet Packages

| Package | Purpose |
|---|---|
| `OpenTelemetry.Extensions.Hosting` | Wires OpenTelemetry into the .NET host lifecycle |
| `OpenTelemetry.Instrumentation.AspNetCore` | Auto-instruments incoming HTTP requests |
| `OpenTelemetry.Instrumentation.Http` | Auto-instruments outgoing HTTP client calls |
| `OpenTelemetry.Instrumentation.EntityFrameworkCore` | Auto-instruments EF Core database commands |
| `OpenTelemetry.Instrumentation.StackExchangeRedis` | Auto-instruments Redis operations |
| `Npgsql.OpenTelemetry` | Auto-instruments raw Npgsql (Dapper) queries |
| `OpenTelemetry.Exporter.OpenTelemetryProtocol` | Exports spans to any OTLP-compatible backend (Jaeger) |

All packages are referenced in `src/Common/Evently.Common.Infrastructure/`.

---

## Configuring in Code

The service name is defined in a single static class so it never drifts between the registration call and any future telemetry tooling.

`src/API/Evently.Api/OpenTelemetry/DiagnosticsConfig.cs`

```csharp
namespace Evently.Api.OpenTelemetry;

public static class DiagnosticsConfig
{
    public const string ServiceName = "Evently.Api";
}
```

---

OpenTelemetry is registered in shared infrastructure so every module inherits instrumentation automatically.

`src/Common/Evently.Common.Infrastructure/InfrastructureConfiguration.cs`

```csharp
services
    .AddOpenTelemetry()
    .ConfigureResource(resource => resource.AddService(serviceName))
    .WithTracing(tracing =>
    {
        tracing
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddEntityFrameworkCoreInstrumentation()
            .AddRedisInstrumentation()
            .AddNpgsql()
            .AddSource(MassTransit.Logging.DiagnosticHeaders.DefaultListenerName);

        tracing.AddOtlpExporter();
    });
```

-> `AddSource(MassTransit.Logging.DiagnosticHeaders.DefaultListenerName)` captures MassTransit message activity so integration event spans appear in the same trace.

-> `AddOtlpExporter()` reads the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable at runtime, so no code change is needed to switch targets across environments.

---

The **MediatR pipeline behavior** tags each active span with the originating module and command or query name before the handler runs.

`src/Common/Evently.Common.Application/Behaviors/RequestLoggingPipelineBehavior.cs`

```csharp
Activity.Current?.SetTag("request.module", moduleName);
Activity.Current?.SetTag("request.name", requestName);
```

These tags appear on the span created by ASP.NET Core instrumentation, so every handler invocation is searchable by module in Jaeger.

---

`DiagnosticsConfig.ServiceName` is passed into `AddInfrastructure` so the OTLP resource name matches the service identifier declared in code.

`src/API/Evently.Api/Program.cs`

```csharp
builder.Services.AddInfrastructure(
    DiagnosticsConfig.ServiceName,
    ...
);

...

app.UseLogContextTraceLogging();
```

---

`LogContextTraceLoggingMiddleware` bridges the two observability systems by pushing the active trace ID into Serilog's log context. Every log line emitted during a request then carries the same `TraceId` that Jaeger recorded, enabling correlation across Seq and Jaeger.

`src/API/Evently.Api/Middleware/LogContextTraceLoggingMiddleware.cs`

```csharp
internal sealed class LogContextTraceLoggingMiddleware(RequestDelegate next)
{
    public Task Invoke(HttpContext context)
    {
        string traceId = Activity.Current?.TraceId.ToString();

        using (LogContext.PushProperty("TraceId", traceId))
        {
            return next.Invoke(context);
        }
    }
}
```

-> The middleware is registered before `UseSerilogRequestLogging` so the trace ID is available on every log write for the lifetime of the request.

---

## Docker Infrastructure

The Jaeger all-in-one image exposes three ports: OTLP/gRPC on `4317`, OTLP/HTTP on `4318`, and the query UI on `16686`.

`docker-compose.yml`

```yaml
evently.jaeger:
  image: jaegertracing/all-in-one:latest
  container_name: Evently.Jaeger
  ports:
    - 4317:4317
    - 4318:4318
    - 16686:16686
```

---

## Exporter Configuration

The OTLP endpoint is set per environment. In development it points to the Jaeger container by hostname.

`src/API/Evently.Api/appsettings.Development.json`

```json
"OTEL_EXPORTER_OTLP_ENDPOINT": "http://evently.jaeger:4317"
```

-> This is the standard OTLP environment variable read directly by the exporter SDK, so switching to a different backend in production requires only an environment variable change.
