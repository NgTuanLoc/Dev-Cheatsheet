---
tags: [dotnet, advanced, aspnetcore]
aliases: [IHostedService, BackgroundService, Hangfire, Quartz, Cron Job]
level: Advanced
---

# Background Services

> **One-liner**: For long-running work in an ASP.NET Core or worker app, implement **`IHostedService`** (or its base class **`BackgroundService`**) — for richer scheduling, persistence, and retries reach for **Hangfire** or **Quartz.NET**.

---

## Quick Reference

| Option | Use case |
|--------|----------|
| `BackgroundService` | One long-running loop per service |
| `IHostedService` | Manual `StartAsync`/`StopAsync` lifecycle |
| `PeriodicTimer` (.NET 6+) | Async-aware "every N seconds" |
| `Channel<T>` + worker | In-process queue (see [[09 - Channels and Pipelines]]) |
| **Hangfire** | Persistent job queue, dashboard, recurring jobs (Cron) |
| **Quartz.NET** | Heavyweight scheduler with calendars/triggers/clustering |
| Worker SDK (`dotnet new worker`) | Standalone background app (no HTTP) |
| Windows Service / systemd | Hosting the worker as an OS service |

| Lifecycle hook | Called when |
|----------------|-------------|
| `StartAsync` | Host starting (before app handles requests) |
| `ExecuteAsync` | The "do work" loop in `BackgroundService` |
| `StopAsync` | Host shutting down — graceful drain |
| `IHostApplicationLifetime` events | `ApplicationStarted`, `ApplicationStopping`, `ApplicationStopped` |

---

## Core Concept

A **hosted service** is a long-lived component owned by the .NET Generic Host. ASP.NET Core, console workers, Aspire apps — all run on the host. Register `IHostedService` and the host calls `StartAsync` on boot and `StopAsync` on shutdown (with a configurable timeout).

`BackgroundService` is a base class that wraps `IHostedService` and gives you `ExecuteAsync(CancellationToken stoppingToken)` — your event loop. The token signals "shutting down — wrap up". Most CPU/wait loops use `await Task.Delay(..., stoppingToken)` or `PeriodicTimer`.

For **persistent, durable** jobs (must survive restart, retried on failure, scheduled by cron) use **Hangfire**: it stores jobs in SQL/Redis, has a built-in dashboard, supports `BackgroundJob.Enqueue`, `Schedule`, `RecurringJob`, with retry policies. Quartz.NET is heavier and more flexible (calendars, time zones, listeners) but rarely needed unless you're porting from the Java world.

The shutdown contract is critical: `StopAsync` is called with a timeout (default 30s). Honor the cancellation token, finish what you started, and don't restart work mid-shutdown.

---

## Syntax & API

### BackgroundService
```csharp
public sealed class HeartbeatService(ILogger<HeartbeatService> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
        try
        {
            while (await timer.WaitForNextTickAsync(stoppingToken))
            {
                log.LogInformation("Heartbeat at {Now}", DateTimeOffset.UtcNow);
            }
        }
        catch (OperationCanceledException) { /* shutting down */ }
    }
}

builder.Services.AddHostedService<HeartbeatService>();
```

### Scoped service inside a singleton hosted service
```csharp
public sealed class OrderProcessor(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
        while (await timer.WaitForNextTickAsync(ct))
        {
            await using var scope = scopeFactory.CreateAsyncScope();
            var db = scope.ServiceProvider.GetRequiredService<ShopContext>();
            // ... operate within scope
        }
    }
}
```

### IHostedService — manual lifecycle
```csharp
public sealed class WarmupService(IDistributedCache cache) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        // run quick startup work — DON'T block; long work belongs in BackgroundService
        await cache.GetAsync("warmup-key", ct);
    }

    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

### Worker SDK
```bash
dotnet new worker -n Shop.Worker
```

```csharp
// Program.cs
var host = Host.CreateApplicationBuilder(args);
host.Services.AddHostedService<EmailSender>();
host.Build().Run();
```

### Channel-backed background queue
```csharp
public sealed class EmailQueue
{
    private readonly Channel<EmailMessage> _ch = Channel.CreateBounded<EmailMessage>(1000);
    public ChannelWriter<EmailMessage> Writer => _ch.Writer;
    public ChannelReader<EmailMessage> Reader => _ch.Reader;
}

public sealed class EmailSender(EmailQueue queue, ISmtpClient smtp) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var msg in queue.Reader.ReadAllAsync(ct))
        {
            try { await smtp.SendAsync(msg, ct); }
            catch (Exception ex) { /* log + maybe re-enqueue */ }
        }
    }
}

builder.Services.AddSingleton<EmailQueue>();
builder.Services.AddHostedService<EmailSender>();
```

### Hangfire
```csharp
builder.Services.AddHangfire(c => c.UsePostgreSqlStorage(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddHangfireServer();

var app = builder.Build();
app.UseHangfireDashboard("/jobs");

// Fire-and-forget
BackgroundJob.Enqueue<IEmailService>(svc => svc.SendAsync("welcome", "alice@x.com", null));

// Delayed
BackgroundJob.Schedule<IEmailService>(svc => svc.SendAsync(...), TimeSpan.FromMinutes(15));

// Recurring
RecurringJob.AddOrUpdate<IReportService>("daily-report", svc => svc.RunAsync(), Cron.Daily);
```

### Quartz.NET
```csharp
builder.Services.AddQuartz(q =>
{
    var key = new JobKey("CleanupJob");
    q.AddJob<CleanupJob>(o => o.WithIdentity(key));
    q.AddTrigger(t => t.ForJob(key).WithCronSchedule("0 0 3 * * ?"));   // 3 AM daily
});
builder.Services.AddQuartzHostedService(o => o.WaitForJobsToComplete = true);

public sealed class CleanupJob(ShopContext db) : IJob
{
    public async Task Execute(IJobExecutionContext ctx) => await db.PurgeExpiredAsync();
}
```

### Graceful shutdown configuration
```csharp
builder.Host.ConfigureHostOptions(o => o.ShutdownTimeout = TimeSpan.FromMinutes(2));
```

### Application lifetime hooks
```csharp
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();
lifetime.ApplicationStarted.Register(() => Console.WriteLine("Up"));
lifetime.ApplicationStopping.Register(() => Console.WriteLine("Draining"));
```

---

## Common Patterns

```csharp
// Pattern: process queue with retry + DLQ
protected override async Task ExecuteAsync(CancellationToken ct)
{
    await foreach (var msg in queue.Reader.ReadAllAsync(ct))
    {
        for (int attempt = 1; attempt <= 5; attempt++)
        {
            try { await Process(msg, ct); break; }
            catch (Exception ex) when (attempt < 5)
            {
                _log.LogWarning(ex, "attempt {N} failed", attempt);
                await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt)), ct);
            }
            catch (Exception ex)
            {
                _log.LogError(ex, "giving up — moving to DLQ");
                await dlq.WriteAsync(msg, ct);
            }
        }
    }
}
```

```csharp
// Pattern: distributed lock for singleton-style work across replicas
public override async Task ExecuteAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        await using var handle = await _lock.AcquireAsync("nightly-job", ct);
        if (handle is null) { await Task.Delay(TimeSpan.FromMinutes(1), ct); continue; }
        await DoNightlyWork(ct);
    }
}
```

```csharp
// Pattern: cron via Hangfire — survives redeploy
RecurringJob.AddOrUpdate<IReports>("monthly", r => r.GenerateAsync(), "0 0 1 * *");
```

---

## Gotchas & Tips

- **`BackgroundService` exceptions kill the host** in .NET 6+ by default — that's a feature (fail fast) but shocking. Wrap your loop in try/catch and log.
- **Hosted services are singletons** — to use scoped services (DbContext, request-scoped), inject `IServiceScopeFactory` and create scopes per iteration.
- **`PeriodicTimer` is the right "do every N seconds"** — `Timer` callbacks aren't async; `Task.Delay` drifts.
- **Honor `stoppingToken`** — pass it through every `await`. Otherwise shutdown stalls until the timeout.
- **Don't run heavy startup in `StartAsync`** — the host blocks on it. Push to `ExecuteAsync`.
- **Multiple replicas + non-idempotent jobs** = duplicate runs. Either (a) run a single replica for that worker, (b) use a distributed lock, (c) use Hangfire/Quartz with cluster awareness.
- **Hangfire dashboard requires authorization** — by default it's accessible to anyone. Add an `IDashboardAuthorizationFilter`.
- **Quartz job DI**: register jobs as transient/scoped services and use `q.UseMicrosoftDependencyInjectionJobFactory()` (now default in newer versions).
- **Long-running tasks block graceful shutdown** — chunk work and check the token between chunks.
- **Don't use `Thread.Sleep` in async services** — blocks the thread pool. Always `await Task.Delay`.
- **Background workers should expose health checks** — add `AddCheck<MyServiceHealth>()` for readiness/liveness probes.

---

## See Also

- [[09 - Channels and Pipelines]]
- [[14 - ASP.NET Core Basics]]
- [[06 - Async and Await]]
- [[18 - Logging]]
