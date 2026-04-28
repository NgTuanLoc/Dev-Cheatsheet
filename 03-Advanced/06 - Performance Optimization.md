---
tags: [dotnet, advanced, performance]
aliases: [BenchmarkDotNet, ArrayPool, ObjectPool, Hot Path]
level: Advanced
---

# Performance Optimization

> **One-liner**: Don't guess — **measure** with BenchmarkDotNet, **profile** the hot path, then attack allocations (`ArrayPool`, `ObjectPool`, `stackalloc`, `Span<T>`), JIT-friendly code, and async overhead.

---

## Quick Reference

| Tool | Purpose |
|------|---------|
| **BenchmarkDotNet** | Microbenchmarks, allocation, JIT/GC stats |
| **dotnet-counters** | Live perf counters (heap, GC, threadpool) |
| **dotnet-trace** | Sampling profiler (CPU + events) → SpeedScope |
| **dotnet-dump** | Process dump for offline analysis |
| **PerfView / dotTrace** | Full GUI profilers (Windows / cross-platform) |
| **MiniProfiler** | Per-request profiler for ASP.NET |
| **ETW / EventPipe** | Low-level event streams |

| Lever | Win |
|-------|-----|
| `ArrayPool<T>.Shared` | Reuse buffers instead of `new byte[N]` |
| `ObjectPool<T>` | Reuse expensive POCOs (e.g. `StringBuilder`) |
| `Span<T>` / `stackalloc` | Zero-allocation slicing/parsing |
| `ValueTask` | Skip `Task` allocation when sync-completed |
| `[StructLayout(LayoutKind.Sequential)]` | Cache-friendly layout |
| `readonly struct` | Avoid defensive copies of value types |
| `[MethodImpl(MethodImplOptions.AggressiveInlining)]` | Hint JIT to inline tight helpers |
| `Server GC` | Throughput on multi-core servers |
| `Tiered Compilation` | Default — JIT recompiles hot code at higher quality |
| **PGO** (Profile-Guided Optimization) | `<TieredPGO>true</TieredPGO>` (.NET 8+ default) |
| **Native AOT** | Ahead-of-time, no JIT, smaller startup |

---

## Core Concept

In .NET, the typical wins are: **fewer allocations** (less GC pressure), **fewer copies** (use `Span<T>`/`in`), **fewer async-state-machine boxings**, and **better data layout** (structs, contiguous arrays).

Always start with a profiler. The bottleneck is rarely where you think — log queries, JSON serialization, exception throwing, and reflection are the common surprises. Optimizing without measuring is a waste of time *and* often slower (cache-unfriendly clever tricks).

The runtime is already very good: tiered JIT recompiles hot methods, PGO uses runtime profiles, the GC is generational and concurrent. Most "perf" wins come from avoiding work — caching, batching, async I/O — not micro-tuning.

---

## Syntax & API

### BenchmarkDotNet
```csharp
// Program.cs
BenchmarkRunner.Run<StringBenches>();

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net90)]
public class StringBenches
{
    private readonly string[] _items = Enumerable.Range(0, 1000).Select(i => $"item-{i}").ToArray();

    [Benchmark(Baseline = true)]
    public string ConcatPlus()
    {
        var s = "";
        foreach (var i in _items) s += i;
        return s;
    }

    [Benchmark]
    public string Builder()
    {
        var sb = new StringBuilder();
        foreach (var i in _items) sb.Append(i);
        return sb.ToString();
    }

    [Benchmark]
    public string JoinAll() => string.Join("", _items);
}
```

```bash
dotnet run -c Release --project Bench
```

### ArrayPool — reuse buffers
```csharp
private static readonly ArrayPool<byte> Pool = ArrayPool<byte>.Shared;

public async Task<int> ReadAndProcessAsync(Stream src, CancellationToken ct)
{
    var buf = Pool.Rent(8192);
    try
    {
        var read = await src.ReadAsync(buf.AsMemory(0, 8192), ct);
        return Process(buf.AsSpan(0, read));
    }
    finally
    {
        Pool.Return(buf, clearArray: true);
    }
}
```

### ObjectPool — reuse expensive objects
```csharp
public sealed class StringBuilderPolicy : IPooledObjectPolicy<StringBuilder>
{
    public StringBuilder Create() => new(256);
    public bool Return(StringBuilder sb) { if (sb.Capacity > 16384) return false; sb.Clear(); return true; }
}

services.AddSingleton<ObjectPool<StringBuilder>>(sp =>
    new DefaultObjectPool<StringBuilder>(new StringBuilderPolicy()));

// Usage
var pool = sp.GetRequiredService<ObjectPool<StringBuilder>>();
var sb = pool.Get();
try { /* use */ }
finally { pool.Return(sb); }
```

### ValueTask — avoid Task allocation
```csharp
public ValueTask<User?> GetAsync(int id)
{
    if (_cache.TryGetValue(id, out var u)) return ValueTask.FromResult(u);   // synchronous
    return new ValueTask<User?>(LoadFromDbAsync(id));                         // falls back to Task
}
```

### Spans for zero-alloc parsing
```csharp
public static int ParseInt(ReadOnlySpan<char> s)
{
    int n = 0;
    foreach (var c in s) n = n * 10 + (c - '0');
    return n;
}

// caller — no string allocation for slice
int year = ParseInt("2026-04-28".AsSpan(0, 4));
```

### Inlining and aggressive JIT hints
```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static int Square(int x) => x * x;
```

### Server GC + Tiered PGO
```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
  <TieredPGO>true</TieredPGO>   <!-- default in .NET 8+ -->
</PropertyGroup>
```

### Profiling commands
```bash
# Live counters
dotnet-counters monitor --process-id 1234 System.Runtime Microsoft.AspNetCore.Hosting

# Trace (CPU sampling)
dotnet-trace collect --process-id 1234 --duration 00:00:30
# → speedscope.com to view

# Dump
dotnet-dump collect -p 1234
dotnet-dump analyze core_dump

# GC logging
DOTNET_GCConcurrent=1 DOTNET_GCServer=1 dotnet run
```

### EF Core — query optimization
```csharp
// Bad — N+1 queries
foreach (var order in db.Orders) Console.WriteLine(order.Customer.Name);

// Good — Include
foreach (var order in db.Orders.Include(o => o.Customer)) Console.WriteLine(order.Customer.Name);

// Better — projection (only what you need)
var rows = db.Orders.Select(o => new { o.Id, o.Customer.Name }).ToList();
```

### Compiled queries (EF Core)
```csharp
private static readonly Func<ShopContext, int, Task<User?>> GetUser =
    EF.CompileAsyncQuery((ShopContext db, int id) => db.Users.FirstOrDefault(u => u.Id == id));

await GetUser(db, 1);   // skips query-tree compilation
```

### System.Text.Json source-generated serializer
```csharp
[JsonSerializable(typeof(User))]
public partial class AppJsonContext : JsonSerializerContext { }

JsonSerializer.Serialize(user, AppJsonContext.Default.User);   // no reflection, AOT-safe
```

---

## Common Patterns

```csharp
// Pattern: cache-aside
public async Task<User?> GetAsync(int id, CancellationToken ct)
{
    var key = $"user:{id}";
    if (_cache.TryGetValue<User>(key, out var u)) return u;

    u = await _repo.GetAsync(id, ct);
    if (u is not null) _cache.Set(key, u, TimeSpan.FromMinutes(5));
    return u;
}
```

```csharp
// Pattern: batch DB inserts
db.Orders.AddRange(orders);
await db.SaveChangesAsync();   // one round trip, not N
```

```csharp
// Pattern: avoid LINQ in tight loops on hot paths
// LINQ adds delegate + iterator allocations
int Sum(int[] arr) { int s = 0; for (int i = 0; i < arr.Length; i++) s += arr[i]; return s; }
```

```csharp
// Pattern: sealed + readonly struct for cache-friendly types
public readonly record struct Point3D(float X, float Y, float Z);
public sealed class Mesh { /* ... */ }   // sealed enables devirtualization
```

---

## Gotchas & Tips

- **Always benchmark in Release** — Debug disables optimizations. BenchmarkDotNet refuses Debug builds.
- **Don't optimize on a microbenchmark alone** — JIT, GC, branch predictor behave differently in a real app. Profile end-to-end.
- **Allocations hurt more than CPU on the server** — Gen 2/LOH GCs stop the world (less so with Background GC, but still). `[MemoryDiagnoser]` + zero-alloc target is a good north star for hot paths.
- **`ValueTask` has rules** — don't `await` it twice, don't pass it around. It's an optimization for sync-fast-path methods.
- **`StringBuilder` is overkill for ≤4 concatenations** — `string.Concat` or interpolation is faster.
- **Reflection is slow** — cache `MethodInfo`, use `Expression.Compile` for fast invokers, or replace with source generators.
- **`async` adds ~80 bytes per call** for state machine — fine, but in a tight inner loop prefer sync.
- **`HttpClient` should be reused** — use `IHttpClientFactory`. Constructing per-request leaks sockets and slows TLS handshake.
- **JSON is the most common bottleneck** in APIs — use System.Text.Json (faster than Newtonsoft) and source generators when you can.
- **Always measure tail latency (p95/p99)**, not just average. Background GCs and lock contention show up in the tail.
- **Premature optimization is the root of all evil** — Knuth said it for a reason. Profile first.

---

## See Also

- [[09 - Memory Management and GC]]
- [[07 - Memory Leaks and Profiling]]
- [[08 - Span and Memory Types]]
- [[11 - Caching]]
