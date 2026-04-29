---
tags: [dotnet, intermediate, aspnetcore]
aliases: [ILogger, Structured Logging, Serilog]
level: Intermediate
---

# Logging

> **One-liner**: `ILogger<T>` is the standard logging abstraction; structured logging with named placeholders (`{User}` not `$"{user}"`) lets you query logs by field — Serilog/OpenTelemetry plug in as providers.

---

## Quick Reference

| Level | Meaning | Default in prod |
|-------|---------|-----------------|
| `Trace` | very fine-grained | off |
| `Debug` | dev diagnostic | off |
| `Information` | normal operation | on |
| `Warning` | unexpected but handled | on |
| `Error` | request-level failure | on |
| `Critical` | service-level failure | on |
| `None` | suppress | — |

| API | Use |
|-----|-----|
| `ILogger<T>` | inject typed logger |
| `LogInformation("X {Y}", y)` | structured log |
| `LogError(ex, "...")` | exception with message |
| `BeginScope(state)` | enrich subsequent logs |
| `LoggerMessage` source-gen | high-perf static logger methods |

---

## Core Concept

`ILogger` accepts a **message template** with `{Named}` placeholders, plus the corresponding values. Don't string-interpolate — interpolation hides the template, defeats structured queries (Seq, Elastic, Datadog), and allocates even when the level is disabled.

The category name is taken from the generic argument (`ILogger<UserService>` → `MyApp.Services.UserService`). Filter levels per category via `appsettings.json`.

For high throughput, use **`LoggerMessage` source generators** — they generate strongly-typed methods that avoid boxing and are cheap when filtered out.

**Scopes** add ambient properties to every log inside a `using` — perfect for correlation IDs and request context.

---

## Syntax & API

### Basic logging
```csharp
public class OrderService(ILogger<OrderService> log)
{
    public async Task<Order> PlaceAsync(int userId, OrderDto dto)
    {
        log.LogInformation("Placing order for {UserId} totaling {Total:C}",
            userId, dto.Total);

        try
        {
            return await ProcessAsync(userId, dto);
        }
        catch (Exception ex)
        {
            log.LogError(ex, "Failed to place order for {UserId}", userId);
            throw;
        }
    }
}
```

### Configure via appsettings.json
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "MyApp.Services": "Debug"
    },
    "Console": {
      "FormatterName": "json"
    }
  }
}
```

### Scopes
```csharp
public async Task<IResult> Handle(HttpContext ctx)
{
    using var scope = _log.BeginScope(new Dictionary<string, object>
    {
        ["RequestId"] = ctx.TraceIdentifier,
        ["UserId"]    = ctx.User.GetId()
    });

    _log.LogInformation("Processing request");
    // every log inside this using will include RequestId + UserId
}
```

### LoggerMessage source generator (.NET 6+)
```csharp
public static partial class Log
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Information,
                   Message = "User {UserId} placed order {OrderId} for {Total:C}")]
    public static partial void OrderPlaced(ILogger logger, int userId, int orderId, decimal total);

    [LoggerMessage(EventId = 5001, Level = LogLevel.Error,
                   Message = "Failed to charge {UserId}")]
    public static partial void ChargeFailed(ILogger logger, Exception ex, int userId);
}

// Usage
Log.OrderPlaced(_log, userId, order.Id, order.Total);
Log.ChargeFailed(_log, ex, userId);
```

### Configure providers
```csharp
builder.Logging.ClearProviders();
builder.Logging.AddConsole();        // human-readable
builder.Logging.AddJsonConsole();    // JSON for log aggregators
builder.Logging.AddDebug();
builder.Logging.AddEventLog();       // Windows
```

### Serilog (popular third-party)
```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Seq
```

```csharp
using Serilog;

builder.Host.UseSerilog((ctx, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.Seq("http://localhost:5341"));
```

```json
"Serilog": {
  "MinimumLevel": "Information",
  "Override": { "Microsoft": "Warning" }
}
```

### OpenTelemetry (logs + metrics + traces)
```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

```csharp
builder.Logging.AddOpenTelemetry(o =>
{
    o.IncludeFormattedMessage = true;
    o.IncludeScopes = true;
    o.AddOtlpExporter();   // ships to Tempo/Jaeger/Datadog/etc.
});
```

---

## Common Patterns

```csharp
// Pattern: don't log + throw — pick one
// Bad — logs in N layers, polluting output
catch (Exception ex)
{
    _log.LogError(ex, "...");
    throw;
}
// Good — let the top-level handler log once
// Or log if you ARE the top — but don't both throw AND log at every layer
```

```csharp
// Pattern: include correlation ID in every request
app.Use(async (ctx, next) =>
{
    var corr = ctx.Request.Headers["X-Correlation-Id"].FirstOrDefault()
            ?? Guid.NewGuid().ToString();
    using (LogContext.PushProperty("CorrelationId", corr))   // Serilog
    {
        ctx.Response.Headers["X-Correlation-Id"] = corr;
        await next();
    }
});
```

```csharp
// Pattern: avoid expensive arg evaluation when level is off
if (_log.IsEnabled(LogLevel.Debug))
{
    _log.LogDebug("Heavy: {Dump}", ExpensiveDump());
}
```

---

## Gotchas & Tips

- **Never use `string.Format` or `$"..."`** in log calls — pass the template + args separately. `LogInformation($"User {id}")` works, but throws away the structure.
- **Don't log secrets** (passwords, tokens, PII) — even in debug. Sanitize at the source.
- **Stack traces are huge** — log exceptions only at the boundary that handles them. Repeated `_log.LogError(ex, ...)` and re-throw doubles output.
- **Async exceptions wrapped in AggregateException** — `LogError(ex, ...)` includes inner; for `Task.WhenAll`, iterate `ex.InnerExceptions`.
- **Console logger is not for production volume** — use a sink that batches (Seq, Elastic, Datadog). Console with thousands of logs/sec becomes the bottleneck.
- **`LoggerMessage` source-gen avoids boxing** — for hot logs (every request), use it instead of `LogInformation`.
- **`BeginScope` returns `IDisposable`** — always wrap in `using` or scope leaks across requests.
- **Filter Microsoft logs to Warning** in production — the framework is chatty at Information.

---

## See Also

- [[17 - Configuration]]
- [[14 - ASP.NET Core Basics]]
- [[19 - Middleware]]
- [[15 - Source Generators]]
