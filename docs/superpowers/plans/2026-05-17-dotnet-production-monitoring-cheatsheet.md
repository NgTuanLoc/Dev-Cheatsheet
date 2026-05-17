# .NET Production Monitoring and Diagnostics Cheatsheet Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a single comprehensive Advanced-level cheatsheet to the `.NET` topic that teaches an engineer how to monitor a production .NET app, recognize the most common failure patterns from telemetry, diagnose them with the right tools, and apply the proven fix.

**Architecture:** One large but focused note at `Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md`. The note organizes the material into three layers: (1) the **observability stack** an engineer should have in place before an incident, (2) a **diagnostic loop** for navigating an active incident, and (3) a **pattern catalog** of ~10 common production failures, each presented as **Symptom → Tool → Root Cause → Fix**. The note complements the existing `18 - Logging`, `07 - Memory Leaks and Profiling`, and `06 - Performance Optimization` notes; it does not duplicate them — it cross-links.

**Tech Stack:** Markdown + YAML frontmatter, Mermaid for diagrams, code fences in `csharp` / `bash` / `json` / `yaml` per `Dotnet/CLAUDE.md`. Telemetry stack covered: OpenTelemetry (.NET SDK), Serilog (structured logging), ASP.NET Core HealthChecks, `dotnet-counters` / `dotnet-trace` / `dotnet-dump` / `dotnet-gcdump`, Application Insights, Prometheus + Grafana, Seq.

---

## Scope Check

This is a single cohesive note (a cheatsheet), not a multi-subsystem deliverable. One plan is correct. The note is large (target ~500–700 lines) because the pattern catalog is the deliverable's main value — but its structure is the canonical template, so the engineer keeps a steady cadence section by section.

---

## File Structure

Files to **create**:

| Path | Responsibility |
|------|----------------|
| `Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md` | The full cheatsheet: observability stack, diagnostic loop, pattern → diagnosis → fix catalog |

Files to **modify**:

| Path | Change |
|------|--------|
| `Dotnet/00-Index/Dotnet Master Index.md` | Add new row under a new "Production operations" sub-section in "03 — Advanced" |
| `Dotnet/00-Index/Dotnet Learning Path.md` | Insert a new "Stage 10 — Production Operations" after Stage 9 referencing this note |
| `Dotnet/CLAUDE.md` | Update advanced count `18 → 19`, append row 19 to advanced coverage map, update total `48 → 49`, add diagram-coverage entry |

No deletes. No existing files restructured beyond these additions. No tests — Obsidian docs, validated by template-conformance and link-resolution greps.

---

## Authoring Conventions (read once, apply to every section)

Internalize before writing:

1. **Canonical section order**: frontmatter → `# Title` → one-liner blockquote → `---` → `## Quick Reference` → `---` → `## Core Concept` (≤300 words) → `---` → `## Diagram` (Mermaid) → `---` → `## Syntax & API` → `---` → `## Common Patterns` → `---` → `## Gotchas & Tips` → `---` → `## See Also`.
2. **Code fence tags**: `csharp`, `bash`, `json`, `xml`, `yaml`, `mermaid`. No bare opening fences. (Closing fences are bare three backticks — that's normal.)
3. **Wiki-links**: `[[Note Title]]` only — no path, no `.md`. Required cross-links from this note (must resolve to existing files): `[[18 - Logging]]`, `[[07 - Memory Leaks and Profiling]]`, `[[06 - Performance Optimization]]`, `[[06 - Async and Await]]`, `[[09 - Memory Management and GC]]`, `[[16 - Entity Framework Core]]`, `[[11 - Caching]]`, `[[10 - Parallel and Dataflow]]`, `[[17 - Docker and Containers]]`, `[[18 - CI-CD and DevOps]]`, `[[08 - Exception Handling]]`.
4. **Style benchmark**: match the depth, density, and code-example richness of `Database/02-Intermediate/21 - B-Tree Internals.md` (~415 lines) and `Dotnet/03-Advanced/07 - Memory Leaks and Profiling.md`. The pattern catalog should read like a senior engineer's incident-response runbook, not a tutorial.
5. **No emojis** in the note body.
6. **Examples must be minimal and compilable** — paste into a fresh ASP.NET Core 8/9 project and run.

---

## The Pattern Catalog — Authoritative Source of Truth

The Common Patterns section is the deliverable's center of gravity. **Each pattern uses the same four-part sub-section template** so a reader scans them identically:

````markdown
### Pattern N: [Pattern name]

**Symptoms** (what you see in telemetry):
- bullet of dashboard/log signal
- bullet of dashboard/log signal

**Confirm with**:
```bash
# concrete command(s) — dotnet-counters / dotnet-trace / log query / etc.
```

**Root causes**:
- bullet of likely cause + brief why
- bullet of likely cause + brief why

**Fix**:
```csharp
// concrete code or config change
```
plus a one-paragraph explanation of *why* the fix works.
````

**The 10 patterns to cover (in this exact order):**

1. **High request latency (P95/P99 spike)**
2. **Memory growth / managed leak**
3. **CPU spike sustained at ~100%**
4. **Thread pool starvation**
5. **Database connection pool exhaustion**
6. **Deadlock (async or `lock`-based)**
7. **GC pressure (frequent Gen 2 collections, LOH growth)**
8. **5xx error spike from unhandled exceptions**
9. **Cold-start latency (slow first request after deploy/scale-out)**
10. **Log flooding / disk I/O saturation from runaway logging**

Content guidance per pattern is embedded in Task 1 Step 1.7 below. **No pattern is "similar to" another — each one must be written out fully**, because the engineer may scan the note non-linearly.

---

## Task 1: Create `19 - Production Monitoring and Diagnostics.md`

**Files:**
- Create: `Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md`
- Reference: `_Templates/Note Template.md`, `Dotnet/CLAUDE.md`, `Database/02-Intermediate/21 - B-Tree Internals.md` (depth benchmark), `Dotnet/03-Advanced/07 - Memory Leaks and Profiling.md` (style benchmark)

### Step 1.1: Frontmatter + title + one-liner

- [ ] **Write the file's first 11 lines exactly:**

```markdown
---
tags: [dotnet, advanced, performance, devops, async, memory]
aliases: [Production Monitoring, Observability, Diagnostics Playbook, Incident Response]
level: Advanced
---

# Production Monitoring and Diagnostics

> **One-liner**: Production .NET incidents follow a small number of repeatable patterns; the job is to (1) have logs / metrics / traces wired up *before* the incident, (2) drive a tight observe → hypothesize → instrument → fix loop *during* it, and (3) recognize the pattern fast enough that the fix is the *known* fix.

---
```

### Step 1.2: Write the Quick Reference section

- [ ] **Two tables. Use these starters exactly; extend rows if useful but do not shorten:**

```markdown
## Quick Reference

### Symptom → first-look tool

| Symptom | First tool to reach for | Then |
|---------|-------------------------|------|
| Latency spike on one endpoint | APM trace (App Insights / Jaeger) for a slow request | Look for the longest span; check downstream dependency |
| Memory growing forever | `dotnet-counters monitor` → `gc-heap-size` | `dotnet-gcdump collect`, diff two snapshots |
| CPU pinned at 100% | `dotnet-counters monitor` → `cpu-usage` + `threadpool-thread-count` | `dotnet-trace collect --profile cpu-sampling` |
| Requests queue / timeouts under load | `threadpool-queue-length`, `threadpool-thread-count` | Check for sync-over-async (`.Result`, `.Wait()`) |
| 500s after a deploy | Logs filtered by `level=Error`, last 15 min | Check exception type; correlate with the deploy SHA |
| EF query slow | `Microsoft.EntityFrameworkCore.Database.Command` log at `Information` | Inspect generated SQL; `EXPLAIN` it; add index |
| One pod crashes repeatedly | `kubectl logs --previous`, exit code, OOMKilled? | Check container memory limit vs heap; dump on crash |
| Cold start slow | App Insights "Server response time" first request after scale-out | Pre-warm, ReadyToRun, AOT, smaller image |

### Tool → primary use

| Tool | Use |
|------|-----|
| **OpenTelemetry** (`OpenTelemetry.Extensions.Hosting`) | Logs + metrics + traces, vendor-neutral, OTLP export |
| **Serilog** (`Serilog.AspNetCore`) | Structured logging with sinks (Seq, Elasticsearch, file) |
| **Application Insights** (`Azure.Monitor.OpenTelemetry.AspNetCore`) | Azure-hosted APM: live metrics, end-to-end traces, profiler |
| **Prometheus + Grafana** | OSS metrics scrape + dashboards; pair with OTel Prom exporter |
| **Seq** | Local/self-hosted structured-log query UI; great for dev/staging |
| **`dotnet-counters`** | Live counter monitor — no instrumentation needed |
| **`dotnet-trace`** | EventPipe collection — CPU sampling, GC events, async causality |
| **`dotnet-dump`** | Full process dump for offline analysis with SOS / WinDbg |
| **`dotnet-gcdump`** | Lightweight heap snapshot — diff to find retention paths |
| **`dotnet-stack`** | Print managed call stacks of all threads — fast deadlock triage |
| **PerfView** (Windows) | ETW traces, allocation profiling, contention analysis |
| **JetBrains dotMemory / dotTrace** | GUI heap/CPU profilers; remote-attach capable |
| **`/health` endpoints** (`Microsoft.Extensions.Diagnostics.HealthChecks`) | Liveness + readiness probes for orchestrators |

---
```

### Step 1.3: Write the Core Concept

- [ ] **Constraint: ≤ 300 words, short paragraphs, no bullets.** Cover in order:

1. **The three pillars of observability** — logs, metrics, traces. Logs answer "what happened?"; metrics answer "how much / how often?"; traces answer "where did the time go in this single request?".
2. **The diagnostic loop** — Observe (dashboard, alert) → Hypothesize (which pillar narrows it?) → Instrument (counter, trace, dump) → Fix → Verify. The loop is fast when you've wired up the pillars *before* the incident; without them you spend the incident fighting visibility, not the bug.
3. **Pattern recognition is the senior-engineer multiplier** — most incidents are a small set of recurring shapes (latency spike, memory growth, CPU spike, thread starvation, connection exhaustion, deadlock, GC pressure, 5xx burst, cold start, log flood). Each has a near-canonical symptom signature and a near-canonical fix. Reading the catalog below is faster than re-deriving each one at 2 AM.
4. **What goes wrong without it** — teams without telemetry guess; teams with telemetry but no pattern vocabulary thrash; teams with both ship the fix in minutes. The remainder of this note builds out both.

End with one sentence linking forward: *"The rest of this note is the cheat-sheet form of that vocabulary."*

### Step 1.4: Write the Diagram section

- [ ] **One Mermaid `graph TD` diagram showing the observability stack: app → telemetry SDK → exporters → backends → dashboards/alerts. Use this exact diagram (do not rename nodes — labels are deliberately specific):**

```markdown
## Diagram

```mermaid
graph TD
    App[ASP.NET Core app<br/>Program.cs]

    subgraph SDK[Telemetry SDKs in-process]
        ILogger[Microsoft.Extensions.Logging<br/>+ Serilog]
        OTel[OpenTelemetry SDK<br/>Tracing + Metrics]
        HC[HealthChecks middleware<br/>/health, /health/ready]
    end

    App --> ILogger
    App --> OTel
    App --> HC

    ILogger --> SerSink[(Serilog sinks)]
    OTel --> OTLP[OTLP exporter]
    OTel --> Prom[Prometheus exporter]

    SerSink --> Seq[(Seq)]
    SerSink --> ES[(Elasticsearch / Loki)]

    OTLP --> AI[(Azure App Insights)]
    OTLP --> Jae[(Jaeger / Tempo)]
    Prom --> PromDB[(Prometheus TSDB)]

    AI --> Dash1[Dashboards + alerts]
    PromDB --> Graf[Grafana]
    Jae --> Graf
    Seq --> Dash2[Log queries]

    HC -.scrape.-> Orch[Orchestrator<br/>K8s liveness/readiness]
\`\`\`
\`\`\`

(Note: when authoring the file, the inner ` ```mermaid ` fence and its closing ` ``` ` are plain three backticks — they are escaped only here in this plan file. Likewise the outer code fence containing the section ends with three plain backticks.)

---
```

### Step 1.5: Write the Syntax & API section — five sub-sections

The Syntax & API section sets up the telemetry stack a reader can paste into a fresh ASP.NET Core 8 / 9 project.

- [ ] **Sub-section A: Wire up Structured Logging with Serilog.** Write this exactly:

````markdown
## Syntax & API

### Structured logging with Serilog

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.Seq
```

```csharp
// Program.cs
using Serilog;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, sp, cfg) => cfg
    .ReadFrom.Configuration(ctx.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithEnvironmentName()
    .WriteTo.Console(outputTemplate:
        "[{Timestamp:HH:mm:ss} {Level:u3}] {SourceContext} {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.Seq("http://seq:5341"));

var app = builder.Build();
app.UseSerilogRequestLogging();   // one log per request, with elapsed ms + status
app.MapGet("/orders/{id:int}", (int id, ILogger<Program> log) =>
{
    log.LogInformation("Fetching order {OrderId}", id);   // structured, NOT $"" interpolation
    return Results.Ok(new { id });
});
app.Run();
```

**Why structured**: each property becomes a queryable column in Seq / Elasticsearch — `OrderId=42` is filterable; a string `"Fetching order 42"` is not. **Never** use string interpolation in `ILogger` calls — you lose the property *and* you incur the format cost even when the log level is filtered out.
````

- [ ] **Sub-section B: OpenTelemetry (traces + metrics).** Write exactly:

````markdown
### OpenTelemetry — traces and metrics

```bash
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package OpenTelemetry.Instrumentation.AspNetCore
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Instrumentation.EntityFrameworkCore
dotnet add package OpenTelemetry.Instrumentation.Runtime
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
```

```csharp
// Program.cs (additive)
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("orders-api", serviceVersion: "1.4.2"))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddEntityFrameworkCoreInstrumentation(o => o.SetDbStatementForText = true)
        .AddOtlpExporter())     // points at OTEL_EXPORTER_OTLP_ENDPOINT
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()    // GC, threadpool, exceptions, contention
        .AddOtlpExporter());
```

```csharp
// Custom span around a critical section
using System.Diagnostics;
private static readonly ActivitySource Activity = new("orders-api");

public async Task<Order> PlaceOrderAsync(OrderRequest req, CancellationToken ct)
{
    using var act = Activity.StartActivity("PlaceOrder");
    act?.SetTag("user.id", req.UserId);
    act?.SetTag("order.total_cents", req.TotalCents);
    // ... business logic ...
    return order;
}
```

**Why both**: traces show *where* time went in one request (a single span tree per HTTP call); metrics show *how much* over time (P99 latency, requests/sec, GC count). Logs without traces = no causality; metrics without logs = no detail.
````

- [ ] **Sub-section C: Health checks for liveness + readiness.** Write exactly:

````markdown
### Health checks — liveness vs readiness

```bash
dotnet add package Microsoft.Extensions.Diagnostics.HealthChecks
dotnet add package AspNetCore.HealthChecks.SqlServer
dotnet add package AspNetCore.HealthChecks.Redis
```

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(builder.Configuration.GetConnectionString("Sql")!, tags: ["ready"])
    .AddRedis(builder.Configuration.GetConnectionString("Redis")!, tags: ["ready"])
    .AddCheck("self", () => HealthCheckResult.Healthy(), tags: ["live"]);

app.MapHealthChecks("/health/live",  new() { Predicate = c => c.Tags.Contains("live")  });
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

```yaml
# Kubernetes Deployment fragment
livenessProbe:
  httpGet: { path: /health/live,  port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /health/ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 2
```

**Liveness** = "is the process alive enough to keep?" — fail → restart. **Readiness** = "should we route traffic to it?" — fail → remove from load-balancer rotation but **don't** restart. Confusing them causes restart storms (readiness failing → liveness restarts → readiness still failing → loop).
````

- [ ] **Sub-section D: Live diagnostics with the `dotnet-*` global tools.** Write exactly:

````markdown
### Live diagnostics — `dotnet-counters` / `-trace` / `-dump` / `-gcdump`

```bash
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-gcdump
dotnet tool install -g dotnet-stack

# Find the target PID
dotnet-counters ps

# Live counters — System.Runtime + ASP.NET + Kestrel + EF
dotnet-counters monitor -p <PID> --counters System.Runtime,Microsoft.AspNetCore.Hosting,Microsoft.EntityFrameworkCore

# 30-second CPU sampling trace → SpeedScope
dotnet-trace collect -p <PID> --profile cpu-sampling --duration 00:00:30 --format Speedscope

# Heap snapshot (no full dump — fast, smaller)
dotnet-gcdump collect -p <PID> -o before.gcdump
# ... reproduce workload ...
dotnet-gcdump collect -p <PID> -o after.gcdump
# Open both in Visual Studio or PerfView; "Diff" view shows new retainers

# Full process dump (for SOS / WinDbg / dotnet-dump analyze)
dotnet-dump collect -p <PID> -o crash.dmp
dotnet-dump analyze crash.dmp
> threads
> clrstack
> dumpheap -stat
> gcroot <address>

# All managed call stacks — instant deadlock triage
dotnet-stack report -p <PID>
```

Running inside a container? Mount `/tmp` shared with the host or use `kubectl exec -it <pod> -- bash` and install the tool there. The tools attach via the **diagnostic port** (a Unix socket at `/tmp/dotnet-diagnostic-<pid>`), so no debug build, no source code, no app restart is required.
````

- [ ] **Sub-section E: Application Insights drop-in (Azure shortcut).** Write exactly:

````markdown
### Application Insights — Azure drop-in

```bash
dotnet add package Azure.Monitor.OpenTelemetry.AspNetCore
```

```csharp
builder.Services.AddOpenTelemetry().UseAzureMonitor(o =>
{
    o.ConnectionString = builder.Configuration["AppInsights:ConnectionString"];
});
```

You now get: distributed traces (per request, per dependency), Live Metrics Stream (real-time), Failures + Performance blades, Profiler (production CPU/allocation traces), Snapshot Debugger (exception state on first occurrence). Sampling defaults to adaptive — turn it down to `1.0` for low-traffic apps so you don't miss the one error you care about.

---
````

### Step 1.6: Write the Common Patterns section — the catalog

The catalog is the deliverable's core. Use the **exact four-part template** (Symptoms / Confirm with / Root causes / Fix) for every single pattern. Numbering: `### Pattern 1: …`, `### Pattern 2: …`, etc.

- [ ] **Pattern 1 — High request latency (P95/P99 spike).** Write fully (not "similar to anywhere"):

````markdown
## Common Patterns

### Pattern 1 — High request latency (P95/P99 spike)

**Symptoms**:
- P95 / P99 latency rises while P50 stays flat → only the slow tail moves
- A single endpoint or a single downstream dependency dominates
- Traces show one long span that wasn't there yesterday

**Confirm with**:
```bash
# In the APM (App Insights / Jaeger / Tempo): filter slow requests
#   duration > P95_threshold AND timestamp > 15m ago
# Inspect the longest span — is it inside your code, in an HTTP client call,
# or in EF Core (look for the SQL in the span tags)?

# Locally on the host:
dotnet-counters monitor -p <PID> \
  --counters Microsoft.AspNetCore.Hosting[total-requests,request-duration]
```

**Root causes**:
- An N+1 query in EF Core (one query becomes N queries inside a loop)
- A downstream HTTP call newly slow (vendor latency, regional failover)
- Synchronous I/O on the request thread (`.Result`, `.Wait()`, sync file/DB calls)
- Lock contention on a hot shared resource
- Cold cache after a deploy / scale-out

**Fix** — depends on the cause; the two most common:

```csharp
// N+1 → use Include / projection
var orders = await db.Orders
    .Include(o => o.Items).ThenInclude(i => i.Product)
    .Where(o => o.UserId == userId)
    .AsNoTracking()
    .ToListAsync(ct);

// Slow HTTP dependency → add a strict per-call timeout + circuit breaker (Polly)
services.AddHttpClient<PricingClient>(c => c.Timeout = TimeSpan.FromSeconds(2))
    .AddStandardResilienceHandler(o =>
    {
        o.CircuitBreaker.FailureRatio = 0.5;
        o.Retry.MaxRetryAttempts = 2;
    });
```

The fix works because the latency budget is now bounded: an unhealthy dependency fails fast and trips the breaker instead of holding request threads. See `[[16 - Entity Framework Core]]` for query patterns and `[[11 - Caching]]` for cache-aside that often eliminates the read path entirely.
````

- [ ] **Pattern 2 — Memory growth / managed leak.** Write fully:

````markdown
### Pattern 2 — Memory growth / managed leak

**Symptoms**:
- Process working set grows monotonically; never returns after traffic drops
- `gc-heap-size` (Gen 2) rises; `gen-2-gc-count` rises but bytes don't recover
- Container eventually OOMKilled; on bare metal, swapping starts

**Confirm with**:
```bash
dotnet-counters monitor -p <PID> --counters System.Runtime
# Watch: gc-heap-size, gen-2-size, gen-2-gc-count, alloc-rate

# Snapshot, reproduce workload, snapshot again, diff:
dotnet-gcdump collect -p <PID> -o before.gcdump
# (drive the workload)
dotnet-gcdump collect -p <PID> -o after.gcdump
# Open both in Visual Studio "Diagnostic Tools" → "Memory" → compare
```

**Root causes**:
- Event handlers never unsubscribed (publisher keeps subscribers alive)
- Static collection growing forever (cache without eviction)
- Captured `this` in long-lived lambda or `Task`
- `HttpClient` created per request (socket exhaustion + memory)
- EF `DbContext` lifetime too long (tracking many entities)
- Uncancelled `Task.Delay` / `Timer` keeping closures alive

**Fix**:
```csharp
// Unsubscribe on dispose
public sealed class OrderView : IDisposable
{
    private readonly OrderService _svc;
    public OrderView(OrderService svc) { _svc = svc; _svc.Changed += OnChanged; }
    private void OnChanged(object? s, EventArgs e) { /* ... */ }
    public void Dispose() => _svc.Changed -= OnChanged;
}

// Bounded cache instead of static Dictionary
services.AddMemoryCache(o => o.SizeLimit = 100_000);

// IHttpClientFactory instead of `new HttpClient()`
services.AddHttpClient<PricingClient>();
```

Most leaks are reference-chain bugs, not allocation bugs. The fix is to break the chain. See `[[07 - Memory Leaks and Profiling]]` for the full diagnosis flow and the four canonical leak shapes.
````

- [ ] **Pattern 3 — CPU spike sustained at ~100%.** Write fully:

````markdown
### Pattern 3 — CPU spike sustained at ~100%

**Symptoms**:
- `cpu-usage` counter pegged
- Latency rises across all endpoints (not one)
- Sometimes throughput collapses (work piles up faster than CPU drains it)

**Confirm with**:
```bash
dotnet-trace collect -p <PID> --profile cpu-sampling --duration 00:00:30 \
    --format Speedscope -o cpu.speedscope.json
# Open https://www.speedscope.app and load the file.
# The widest bar is the hottest method — that's where to look.
```

**Root causes**:
- An accidental tight loop (missing `await`, retry without back-off, recursion)
- Regex catastrophic backtracking on user input
- JSON serialization on a massive object on the hot path
- Logging at `Debug` in production (`if (log.IsEnabled(LogLevel.Debug))` guard missing)
- A GC stuck collecting a huge heap (different shape — see Pattern 7)

**Fix**:
```csharp
// Compile regex; set a timeout to defeat catastrophic backtracking
private static readonly Regex Email = new(
    @"^[^@\s]+@[^@\s]+\.[^@\s]+$",
    RegexOptions.Compiled | RegexOptions.CultureInvariant,
    matchTimeout: TimeSpan.FromMilliseconds(50));

// Source-generated JSON serializer instead of reflection
[JsonSerializable(typeof(OrderDto))]
internal partial class AppJsonContext : JsonSerializerContext;

// Use the context — zero reflection, faster, AOT-safe
JsonSerializer.Serialize(order, AppJsonContext.Default.OrderDto);

// Guard expensive log construction
if (log.IsEnabled(LogLevel.Debug))
    log.LogDebug("Payload {Payload}", JsonSerializer.Serialize(req));
```

CPU spikes are usually a hot path doing more work per call, not more calls. Find it with sampling. See `[[06 - Performance Optimization]]` for BenchmarkDotNet workflows and `[[15 - Source Generators]]` for source-gen serializers.
````

- [ ] **Pattern 4 — Thread pool starvation.** Write fully:

````markdown
### Pattern 4 — Thread pool starvation

**Symptoms**:
- `threadpool-queue-length` climbs (pending work outpaces threads)
- `threadpool-thread-count` rises slowly (one thread per ~500 ms via hill-climbing)
- Latency spikes across *all* endpoints, including endpoints doing no work
- Health checks start failing under load even though the host is idle

**Confirm with**:
```bash
dotnet-counters monitor -p <PID> --counters System.Runtime[\
threadpool-queue-length,\
threadpool-thread-count,\
threadpool-completed-items-count]

# Or take a stack snapshot — many threads stuck inside Monitor.Wait or .Result
dotnet-stack report -p <PID> | grep -E "Wait|Result"
```

**Root causes**:
- Sync-over-async: `.Result`, `.Wait()`, `Task.Run(async () => ...).Result` on async APIs
- `ConfigureAwait(false)` missing in a library that's called from a sync caller
- Synchronous I/O (sync file/database call) on async request paths
- Misuse of `Task.Run` for I/O-bound work (spends threads, gains nothing)

**Fix**:
```csharp
// BAD — blocks a threadpool thread inside an async call
public IActionResult GetThing(int id) => Ok(_svc.GetAsync(id).Result);

// GOOD — async all the way down
public async Task<IActionResult> GetThing(int id, CancellationToken ct)
    => Ok(await _svc.GetAsync(id, ct));

// Raise the minimum thread count for bursty workloads with proven need:
//   In Program.cs, BEFORE host.Run():
ThreadPool.SetMinThreads(workerThreads: 200, completionPortThreads: 200);
```

The root cause is always one of "we wait for I/O on a CPU thread" or "we wait for an async result synchronously". See `[[06 - Async and Await]]`. Setting `SetMinThreads` is a *workaround*, not a fix — find the sync-over-async first.
````

- [ ] **Pattern 5 — Database connection pool exhaustion.** Write fully:

````markdown
### Pattern 5 — Database connection pool exhaustion

**Symptoms**:
- Bursts of `InvalidOperationException: Timeout expired. The timeout period elapsed prior to obtaining a connection from the pool.`
- Latency rises with load; some requests succeed, others time out
- `Microsoft.Data.SqlClient` event-counter `active-hard-connections` at the pool ceiling

**Confirm with**:
```bash
dotnet-counters monitor -p <PID> --counters Microsoft.Data.SqlClient.EventSource
# Watch: active-hard-connections, hard-connects, soft-connects

# Check the connection string for an explicit Max Pool Size
# Default is 100 — easy to hit
```

**Root causes**:
- A `DbContext` / `SqlConnection` not disposed (held until GC)
- A long-running transaction holding a connection while doing other work
- `DbContext` registered as `Singleton` (must be `Scoped`)
- `Max Pool Size` too low for the workload
- Connection leaks under exception paths (no `using`)

**Fix**:
```csharp
// EF: ensure Scoped (this is AddDbContext's default — don't override)
services.AddDbContext<AppDb>(o => o.UseSqlServer(connStr));

// Always dispose; the `using` declaration is the simplest correct form
public async Task<Order?> GetAsync(int id, CancellationToken ct)
{
    await using var db = await _factory.CreateDbContextAsync(ct);
    return await db.Orders.FindAsync([id], ct);
}

// Right-size the pool if your concurrency genuinely needs it:
//   "Server=...;Max Pool Size=200;Connection Lifetime=300;"
```

Pool exhaustion is almost always a *leak* (connections not returned) or a *lifetime* bug (singleton context). The fix is rarely "raise the pool" — that hides the leak. See `[[16 - Entity Framework Core]]` for `DbContext` lifetime rules.
````

- [ ] **Pattern 6 — Deadlock (async or `lock`-based).** Write fully:

````markdown
### Pattern 6 — Deadlock (async or `lock`-based)

**Symptoms**:
- Specific endpoints hang forever; others continue
- `dotnet-stack` shows several threads waiting on each other's monitors
- In legacy ASP.NET (not Core), `.Result` deadlocks when context-captured

**Confirm with**:
```bash
# Instant view: dump all stacks and look for the wait chain
dotnet-stack report -p <PID>

# Inside a full dump:
dotnet-dump analyze app.dmp
> threads
> syncblk            # which threads own which managed locks
> clrstack -all      # all managed stacks
```

**Root causes**:
- Locks acquired in inconsistent order across code paths (A→B vs B→A)
- `async` code under a captured `SynchronizationContext` blocking on `.Result`
- A reentrant lock held across an `await` (lock is per-thread; await may resume on another)
- Two `SemaphoreSlim`s with `WaitAsync` in opposite orders

**Fix**:
```csharp
// 1) Define and document a global lock order:
//    Always acquire _userLock BEFORE _orderLock — never reverse.

// 2) Never hold a `lock` across an `await`:
private readonly SemaphoreSlim _gate = new(1, 1);
public async Task UseAsync(CancellationToken ct)
{
    await _gate.WaitAsync(ct);
    try { await DoWorkAsync(ct); }   // OK — semaphore is async-friendly
    finally { _gate.Release(); }
}

// 3) Prefer async coordination primitives over `lock` in async code:
//    SemaphoreSlim, Channel<T>, ReaderWriterLockSlim only sparingly.
```

In .NET Core / ASP.NET Core, the classic `.Result` deadlock is gone — there's no captured `SynchronizationContext` by default. But `SemaphoreSlim` deadlocks and lock-order bugs survive intact. See `[[08 - Synchronization Primitives]]` for the full primitive cheat sheet.
````

- [ ] **Pattern 7 — GC pressure (frequent Gen 2 + LOH growth).** Write fully:

````markdown
### Pattern 7 — GC pressure (frequent Gen 2 + LOH growth)

**Symptoms**:
- `gen-2-gc-count` rising rapidly
- `loh-size` rising
- CPU has spikes correlated with GC pauses (Workstation GC); throughput dips (Server GC)
- `% time in GC` counter > ~10% sustained

**Confirm with**:
```bash
dotnet-counters monitor -p <PID> --counters System.Runtime[\
gc-heap-size,gen-0-gc-count,gen-1-gc-count,gen-2-gc-count,\
loh-size,poh-size,alloc-rate,gc-fragmentation,time-in-gc]

# Allocation trace — which type allocates the most?
dotnet-trace collect -p <PID> --providers Microsoft-Windows-DotNETRuntime:0x1:4 \
    --duration 00:00:30 -o gc.nettrace
# Open in PerfView → "GC Stats" / "GC Heap Net Mem"
```

**Root causes**:
- Allocating large arrays/strings ≥ 85 KB → LOH (collected only on Gen 2)
- Per-request large object reallocation instead of pooling
- Boxing in hot loops (`object[]` of value types, `string.Format`)
- `StringBuilder` not reused; large `byte[]` not pooled
- `Server GC` disabled in a server workload (defaults differ on workstation hosts)

**Fix**:
```csharp
// Pool large buffers
private static readonly ArrayPool<byte> Pool = ArrayPool<byte>.Shared;
public async Task ProcessAsync(Stream s, CancellationToken ct)
{
    byte[] buf = Pool.Rent(128 * 1024);
    try
    {
        int n;
        while ((n = await s.ReadAsync(buf, ct)) > 0) { /* ... */ }
    }
    finally { Pool.Return(buf); }
}

// Use Span<T> / stackalloc for short-lived small buffers
Span<char> tmp = stackalloc char[64];

// Enable Server GC explicitly in csproj if the host hasn't:
//   <PropertyGroup><ServerGarbageCollection>true</ServerGarbageCollection></PropertyGroup>
```

GC isn't slow; allocation is. The fix is to allocate less in the hot path — pool, reuse, or stack-allocate. See `[[09 - Memory Management and GC]]` and `[[08 - Span and Memory Types]]`.
````

- [ ] **Pattern 8 — 5xx error spike from unhandled exceptions.** Write fully:

````markdown
### Pattern 8 — 5xx error spike from unhandled exceptions

**Symptoms**:
- Sudden burst of HTTP 500 responses
- Correlates with a deploy, a feature flag flip, or a downstream change
- App Insights "Failures" blade shows one exception type dominating

**Confirm with**:
```bash
# Filter logs in Seq / Kibana / App Insights:
#   level >= Error AND timestamp > now-30m
#   group by ExceptionType, top 5
# Pull one full exception with stack — fix from the top frame downward.

# Or in code, ensure the exception handler emits structured exception info:
app.UseExceptionHandler();
```

**Root causes**:
- A new code path with a missing null-check (`NullReferenceException`)
- A serialization shape change (mismatched DTO version client ↔ server)
- A migration that hasn't run on this environment (`SqlException: invalid column`)
- A timeout to a downstream dependency (`TaskCanceledException`)
- A `JsonException` from a malformed payload — should be 400, returning 500

**Fix**:
```csharp
// Map known exception types to correct status codes — don't 500 on a 400
app.UseExceptionHandler(eh => eh.Run(async ctx =>
{
    var feature = ctx.Features.Get<IExceptionHandlerFeature>();
    var ex = feature?.Error;
    var (status, code) = ex switch
    {
        JsonException                       => (StatusCodes.Status400BadRequest, "bad_request"),
        TaskCanceledException               => (StatusCodes.Status504GatewayTimeout, "upstream_timeout"),
        ArgumentException                   => (StatusCodes.Status400BadRequest, "bad_request"),
        InvalidOperationException ioe when ioe.Message.Contains("not found")
                                            => (StatusCodes.Status404NotFound, "not_found"),
        _                                   => (StatusCodes.Status500InternalServerError, "internal")
    };
    ctx.Response.StatusCode = status;
    await ctx.Response.WriteAsJsonAsync(new { error = code });
}));
```

A 5xx in the wrong place ruins your alerting signal-to-noise. Map deliberate failures to their real status codes; reserve 5xx for genuinely unexpected exceptions. See `[[08 - Exception Handling]]` for `ProblemDetails` and global handlers.
````

- [ ] **Pattern 9 — Cold-start latency.** Write fully:

````markdown
### Pattern 9 — Cold-start latency (slow first request)

**Symptoms**:
- First request after deploy / scale-out takes seconds (P99 of first-100 requests blows out)
- App Insights "Server response time" first-request chart shows a spike that decays
- Common in serverless (Azure Functions, AWS Lambda), K8s scale-from-zero, and after rolling restarts

**Confirm with**:
```bash
# In your load-test harness, separate "first request after start" from "steady-state":
hey -n 1 -c 1 https://api.example.com/orders/1   # cold
hey -n 1000 -c 50 https://api.example.com/orders/1   # warm

# Inspect the cold path with a startup trace
dotnet-trace collect -p <PID> --profile gc-collect \
    --providers Microsoft-Extensions-Logging \
    --duration 00:00:15
```

**Root causes**:
- JIT compilation of hot paths on first call
- Lazy DI registrations resolving for the first time
- EF Core model warm-up (compiled model not used)
- Reflection-heavy startup (Newtonsoft.Json, AutoMapper config)
- Container image cold (registry pull on scheduler)

**Fix**:
```xml
<!-- csproj: emit ReadyToRun images (pre-JITed) -->
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
  <TieredCompilation>true</TieredCompilation>
  <TieredPGO>true</TieredPGO>
</PropertyGroup>
```

```csharp
// EF Core: precompile the model
services.AddDbContext<AppDb>(o =>
    o.UseSqlServer(connStr)
     .UseModel(AppDbModel.Instance));    // dotnet ef dbcontext optimize

// Touch the hot path during startup to JIT it before traffic arrives
app.Lifetime.ApplicationStarted.Register(async () =>
{
    using var scope = app.Services.CreateScope();
    var svc = scope.ServiceProvider.GetRequiredService<OrdersService>();
    await svc.WarmUpAsync();
});
```

For very strict cold-start budgets, use **Native AOT** — see `[[16 - Native Interop and AOT]]`. For container workloads, also see `[[17 - Docker and Containers]]` (small images = faster pulls).
````

- [ ] **Pattern 10 — Log flooding / disk I/O saturation from runaway logging.** Write fully:

````markdown
### Pattern 10 — Log flooding / disk I/O saturation from runaway logging

**Symptoms**:
- Disk I/O saturated on the host; logs directory grows hundreds of MB / minute
- Network out spikes if logs ship over the wire (Elasticsearch, Loki, App Insights)
- Latency rises because the logger blocks on a full I/O queue

**Confirm with**:
```bash
# Find which log levels are firing how often (Seq query, App Insights logs)
#   count() by SeverityLevel

# On Linux host:
iotop -aoP | head        # who is writing?
ls -lhS /var/log/app | head

# In .NET: are we logging in a tight loop?
dotnet-counters monitor -p <PID> --counters Microsoft.Extensions.Logging
```

**Root causes**:
- A debug log inside a hot loop accidentally promoted to `Information`
- An exception logged at every retry attempt (loop emits N copies)
- Logging full request/response bodies on every request
- A sink misconfigured to write synchronously
- Sampling disabled on App Insights for a high-traffic service

**Fix**:
```csharp
// Log once per failure, not per retry
services.AddHttpClient<X>().AddStandardResilienceHandler(o => {
    o.Retry.OnRetry = args => { /* DON'T log here */ return ValueTask.CompletedTask; };
});
// Log the final failure in the caller.

// Sample noisy categories
"Logging": {
    "LogLevel": {
        "Default": "Information",
        "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
    }
}

// Async + bounded buffer for file sinks (Serilog)
.WriteTo.Async(a => a.File("logs/app-.log",
    rollingInterval: RollingInterval.Day,
    buffered: true,
    flushToDiskInterval: TimeSpan.FromSeconds(1)),
    bufferSize: 10_000,
    blockWhenFull: false)        // drop on overflow, never block app
```

Logs you can't query are noise; logs that block your app are an outage. Tune levels per category, sample at the SDK or sink, and never let logging block business code. See `[[18 - Logging]]` for level/category strategy.

---
````

### Step 1.7: Write the Gotchas & Tips section

- [ ] **Bullet list, 12–15 items, mixing 'before incident' and 'during incident' advice. Required items:**

```markdown
## Gotchas & Tips

- **Wire telemetry up before you need it.** The cheapest moment to add OpenTelemetry / Serilog / health checks is the day you create the project. The most expensive moment is during an outage.
- **Use structured logging — always.** `log.LogInformation("Fetched {Count} orders for {UserId}", orders.Count, userId)` is filterable. `log.LogInformation($"Fetched {orders.Count} orders for {userId}")` is a haystack.
- **Health checks: liveness ≠ readiness.** Liveness fails → restart. Readiness fails → de-route only. Combine them and you'll restart healthy pods just because Redis is slow.
- **Sampling is not a dirty word.** 100% trace retention is unaffordable at scale. Adaptive sampling preserves the rare error trace and the latency tail without flooding the backend.
- **Correlate via `TraceId`.** Add `traceparent` propagation across services; surface `TraceId` in every error response. Without it, "find the related log line" is a needle-in-haystack at 2 AM.
- **One log per request is enough for the happy path.** `UseSerilogRequestLogging()` adds one log per request with duration + status. Inside business logic, log decisions, not narration.
- **Don't `.Result` / `.Wait()` an async call.** It's how you discover thread pool starvation in production. Search for `\.Result` and `\.Wait\(\)` in your codebase periodically.
- **`IHttpClientFactory` exists for a reason.** Direct `new HttpClient()` exhausts sockets *and* misses DNS changes (cached forever). Always go through the factory.
- **The default `DbContext` lifetime is Scoped — keep it.** Singleton `DbContext` is the #1 cause of connection-pool exhaustion and stale-data bugs.
- **`dotnet-counters` is free — install it on every prod host.** Counters add no overhead unless someone is monitoring; they ship inside the runtime; they answer "what is the app doing right now?" in five seconds.
- **Take heap snapshots in pairs.** A single snapshot tells you "what's in memory now"; two snapshots taken around a workload tell you "what *grew*" — the latter is where leaks live.
- **Map exceptions to status codes.** A 5xx is "we broke"; a 4xx is "you sent bad data". Conflating them ruins alert thresholds.
- **Reserve `LogError` for actionable surprises.** If "user typed an invalid email" is logged at Error, alert fatigue kills your team. Use `Warning` or `Information` for expected-bad input.
- **Profile in production-shaped environments.** A profile on your laptop with cached data and no concurrency tells you nothing about the prod hot path. Use App Insights Profiler, dotMemory remote-attach, or a staging environment with a representative load test.
- **Save dumps on crash automatically.** Set `DOTNET_DbgEnableMiniDump=1` and `DOTNET_DbgMiniDumpType=4` (heap dump) on container images you care about — when the next OOM happens you'll have forensic data without re-deploying.

---
```

### Step 1.8: Write the See Also section

- [ ] **Exact list:**

```markdown
## See Also

- [[18 - Logging]] — `ILogger`, log levels, structured logging, Serilog
- [[07 - Memory Leaks and Profiling]] — full diagnosis walkthrough for leaks
- [[06 - Performance Optimization]] — BenchmarkDotNet, ArrayPool, hot paths
- [[06 - Async and Await]] — sync-over-async is the root of half this note
- [[08 - Synchronization Primitives]] — async-friendly coordination
- [[09 - Memory Management and GC]] — Gen 0/1/2, LOH, GC modes
- [[08 - Span and Memory Types]] — zero-allocation hot paths
- [[16 - Entity Framework Core]] — N+1, `AsNoTracking`, DbContext lifetimes
- [[11 - Caching]] — cache-aside, `IMemoryCache`, `IDistributedCache`
- [[08 - Exception Handling]] — global handlers, `ProblemDetails`
- [[17 - Docker and Containers]] — container limits, `DOTNET_DbgEnableMiniDump`
- [[18 - CI-CD and DevOps]] — promoting builds, smoke-test gates, deploy correlation
```

### Step 1.9: Self-check

- [ ] **Run the following verifications:**

1. Section order matches canonical: title → one-liner → Quick Reference → Core Concept → Diagram → Syntax & API → Common Patterns → Gotchas & Tips → See Also.
2. Core Concept ≤ 300 words. Eyeball-count or paste into a counter.
3. **Bare-fence check** — every opening fence has a language tag:

```bash
grep -nE '^```$' "Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md"
```

Every match must be a CLOSING fence (preceded by content, followed by section/prose). No opening fences without a tag.

4. **Wiki-link inventory**:

```bash
grep -oE '\[\[[^]]+\]\]' "Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md"
```

Every link target must correspond to an existing file in `Dotnet/`. Required cross-link targets (all exist):
- `[[18 - Logging]]`, `[[07 - Memory Leaks and Profiling]]`, `[[06 - Performance Optimization]]`, `[[06 - Async and Await]]`, `[[08 - Synchronization Primitives]]`, `[[09 - Memory Management and GC]]`, `[[08 - Span and Memory Types]]`, `[[16 - Entity Framework Core]]`, `[[11 - Caching]]`, `[[08 - Exception Handling]]`, `[[17 - Docker and Containers]]`, `[[18 - CI-CD and DevOps]]`, `[[15 - Source Generators]]`, `[[16 - Native Interop and AOT]]`.

5. All 10 patterns are present, in order, each with all four sub-headings (Symptoms / Confirm with / Root causes / Fix).

### Step 1.10: Commit

- [ ] **Commit ONLY this file:**

```bash
git add "Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md"
git commit -m "docs(dotnet): add 19 - Production Monitoring and Diagnostics note"
```

Do NOT stage anything else. The repo has pre-existing unrelated changes — leave them alone.

---

## Task 2: Update Dotnet Master Index

**Files:**
- Modify: `Dotnet/00-Index/Dotnet Master Index.md`

- [ ] **Step 2.1: Read the file first.**

- [ ] **Step 2.2: Locate the `### Compiler & deployment` sub-section under `## 03 — Advanced (architecture, performance, infra)`. Add a NEW sub-section `### Production operations` AFTER it.** The new sub-section goes immediately after the `### Compiler & deployment` table (which ends with row 18). Insert exactly this block:

```markdown

### Production operations
| # | Note |
|---|------|
| 19 | [[19 - Production Monitoring and Diagnostics]] |
```

- [ ] **Step 2.3: Verify**

```bash
grep -n "19 - Production Monitoring" "Dotnet/00-Index/Dotnet Master Index.md"
```

Expected: exactly one match, immediately under a `### Production operations` heading.

- [ ] **Step 2.4: Commit**

```bash
git add "Dotnet/00-Index/Dotnet Master Index.md"
git commit -m "docs(dotnet): index production monitoring note"
```

---

## Task 3: Update Dotnet Learning Path

**Files:**
- Modify: `Dotnet/00-Index/Dotnet Learning Path.md`

- [ ] **Step 3.1: Read the file first.**

- [ ] **Step 3.2: Append a new "Stage 10 — Production Operations" stage AFTER `## Stage 9 — Compiler & Deployment` and BEFORE the `## Tips for using this path` section.** Insert exactly this block:

```markdown
## Stage 10 — Production Operations (3–5 days)

**Goal**: When the app is in production and something goes wrong, you reach for the right tool first time — not the third time.

1. [[19 - Production Monitoring and Diagnostics]] — symptom → diagnose → fix playbook for the ten patterns you'll actually see

**Checkpoint**: Pick any one production-style app you've built. Wire up Serilog + OpenTelemetry + `/health` + Application Insights (or Prometheus/Grafana) end-to-end, then induce one of the ten patterns and prove you can diagnose it from telemetry alone.

---

```

- [ ] **Step 3.3: Verify**

```bash
grep -n "Stage 9\|Stage 10\|Tips for using" "Dotnet/00-Index/Dotnet Learning Path.md"
```

Expected: `Stage 9` then `Stage 10` then `Tips for using`, in that order.

- [ ] **Step 3.4: Commit**

```bash
git add "Dotnet/00-Index/Dotnet Learning Path.md"
git commit -m "docs(dotnet): add Stage 10 — Production Operations to learning path"
```

---

## Task 4: Update `Dotnet/CLAUDE.md` Coverage Map

**Files:**
- Modify: `Dotnet/CLAUDE.md`

- [ ] **Step 4.1: Read the file first.**

- [ ] **Step 4.2: Update the advanced count.** Find `### Advanced (03-Advanced) — 18 notes` and change to:

```markdown
### Advanced (03-Advanced) — 19 notes
```

- [ ] **Step 4.3: Append one row to the advanced coverage map.** The table currently ends with row 18 (`CI-CD and DevOps`). Add immediately after:

```markdown
| 19 | Production Monitoring and Diagnostics | Observability pillars (logs/metrics/traces), OpenTelemetry, Serilog, health checks, dotnet-counters/trace/dump/gcdump, pattern catalog (latency, leaks, CPU, threadpool, connection pool, deadlock, GC pressure, 5xx, cold start, log flood) |
```

- [ ] **Step 4.4: Update the total line.** Find `**Total: 48 notes** (10 beginner + 20 intermediate + 18 advanced)` and change to:

```markdown
**Total: 49 notes** (10 beginner + 20 intermediate + 19 advanced)
```

- [ ] **Step 4.5: Append a diagram-coverage entry.** In the `## Diagram Coverage Targets` table, append:

```markdown
| Production Monitoring and Diagnostics | graph (observability stack — SDKs → exporters → backends → dashboards) |
```

- [ ] **Step 4.6: Verify**

```bash
grep -nE "Advanced \(03-Advanced\) — 19 notes|Total: 49 notes|19 \| Production Monitoring" "Dotnet/CLAUDE.md"
```

Expected: 3 matching lines.

- [ ] **Step 4.7: Commit**

```bash
git add "Dotnet/CLAUDE.md"
git commit -m "docs(dotnet): update coverage map for production monitoring note"
```

---

## Task 5: Final Cross-Reference Verification

**Files:**
- Modify: none unless fixes needed.

- [ ] **Step 5.1: Confirm all wiki-link targets in the new note resolve.**

```bash
grep -oE '\[\[[^]]+\]\]' "Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md" | sort -u
```

For each unique target, verify a matching `<title>.md` exists somewhere in `Dotnet/`. All required targets correspond to existing files in this vault (see list in Step 1.8).

- [ ] **Step 5.2: Confirm no bare opening fences anywhere in the new file.**

```bash
grep -nE '^```$' "Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md"
```

Every match must be a closing fence. Inspect each line: the line above it should contain content, not a heading. If any match is an *opening* fence (line above is a heading or blank), edit the file to add a language tag.

- [ ] **Step 5.3: Confirm pattern catalog completeness.**

```bash
grep -nE '^### Pattern [0-9]+ —' "Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md"
```

Expected: exactly 10 matches, numbered 1 through 10, in order.

- [ ] **Step 5.4: Confirm the four-part template is used consistently per pattern.**

```bash
grep -nE '^\*\*(Symptoms|Confirm with|Root causes|Fix)\*\*' "Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md"
```

Expected: exactly 40 matches (10 patterns × 4 headings).

- [ ] **Step 5.5: Final commit ONLY if Steps 5.1–5.4 produced fixes.**

```bash
git add "Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md"
git commit -m "docs(dotnet): fix link / fence / pattern-template issues from final review"
```

If no fixes were needed, skip the commit.

---

## Self-Review Notes (from the plan author)

**Spec coverage**: The user asked for a cheatsheet that (a) explains how to monitor a production .NET app, (b) recognizes patterns/bugs/problems, (c) gives solutions. Task 1 Steps 1.5 (Syntax & API — telemetry stack setup) covers (a); Step 1.6 (the 10-pattern catalog with Symptoms / Confirm with / Root causes / Fix) covers (b) and (c) simultaneously. The Quick Reference's symptom→tool table gives readers a 30-second navigation index.

**Placeholders**: None. Every code snippet, command, and table is fully written. The pattern catalog spells out all 10 patterns in full — no "similar to Pattern N" references.

**Type / name consistency**: The file path `Dotnet/03-Advanced/19 - Production Monitoring and Diagnostics.md` is identical in every place it appears (Tasks 1, 2, 4, 5 + commit commands). All cross-linked notes are checked against the actual file glob — they all exist.

**Convention conformance**: Frontmatter tags use only allowed values from `Dotnet/CLAUDE.md` (`dotnet`, `advanced`, `performance`, `devops`, `async`, `memory`). Code-fence languages are limited to `csharp`, `bash`, `json`, `xml`, `yaml`, `mermaid` per topic policy.

**Scope discipline**: The note does NOT re-teach Serilog, EF Core, or async/await — it shows the *production hooks* and cross-links to the existing intermediate notes for foundational depth. This avoids duplication and keeps the cheatsheet scannable.
