---
tags: [dotnet, advanced, performance, aspnetcore]
aliases: [IMemoryCache, IDistributedCache, Redis, HybridCache, Cache-Aside]
level: Advanced
---

# Caching

> **One-liner**: ASP.NET Core ships **`IMemoryCache`** (in-process) and **`IDistributedCache`** (Redis/SQL/etc) — and as of .NET 9, **`HybridCache`** unifies both with stampede protection, tagged invalidation, and built-in serialization.

---

## Quick Reference

| Abstraction | Storage | Use case |
|-------------|---------|----------|
| `IMemoryCache` | in-process RAM | single-instance app, ephemeral state |
| `IDistributedCache` | Redis / SQL / NCache | multi-instance app, shared cache |
| `HybridCache` (.NET 9) | L1 in-memory + L2 distributed | best of both, recommended default |
| Output cache (.NET 7+) | response cache | full HTTP response, per-route |
| Response cache headers | client/CDN | `[ResponseCache]` / Cache-Control |

| Eviction policy | Setting |
|-----------------|---------|
| Absolute expiration | `AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)` |
| Sliding expiration | `SlidingExpiration = TimeSpan.FromMinutes(5)` (resets on access) |
| Size limit | `SizeLimit` on `MemoryCacheOptions`, `Size` per entry |
| Priority | `CacheItemPriority.NeverRemove / High / Normal / Low` |
| Post-eviction callback | `RegisterPostEvictionCallback(...)` |

---

## Core Concept

A cache trades **freshness for speed**. Reads hit memory or a nearby Redis instead of a slow upstream (DB, API). The classic pattern is **cache-aside**: try cache → miss → load → store → return. The risk is **stampede** (cache miss → 100 callers all hit the DB) — solved with locking or `HybridCache`'s built-in protection.

`IMemoryCache` is fastest but per-process: a 4-instance ASP.NET app has 4 separate caches that diverge. `IDistributedCache` (Redis) is shared but adds network latency. **`HybridCache`** layers both — fast in-memory L1 backed by Redis L2 — and handles stampede with `GetOrCreateAsync` that ensures only one factory runs per key.

The hardest part of caching is **invalidation**. Time-based TTL is simple but gives stale reads. Event-based invalidation (publish on update, evict on subscribe) is fresher but harder to wire across services. Tagged invalidation (`HybridCache.RemoveByTag`) makes it tractable.

---

## Syntax & API

### IMemoryCache
```csharp
builder.Services.AddMemoryCache(o => o.SizeLimit = 100_000);

public sealed class UserService(IMemoryCache cache, IUserRepo repo)
{
    public Task<User?> GetAsync(int id, CancellationToken ct) =>
        cache.GetOrCreateAsync($"user:{id}", entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            entry.Size = 1;
            return repo.GetAsync(id, ct);
        });
}
```

### IDistributedCache (Redis)
```csharp
builder.Services.AddStackExchangeRedisCache(o =>
{
    o.Configuration = builder.Configuration.GetConnectionString("Redis");
    o.InstanceName = "shop:";
});

public sealed class CatalogService(IDistributedCache cache, ICatalogRepo repo)
{
    public async Task<Product?> GetAsync(int id, CancellationToken ct)
    {
        var key = $"product:{id}";
        var bytes = await cache.GetAsync(key, ct);
        if (bytes is not null) return JsonSerializer.Deserialize<Product>(bytes);

        var product = await repo.GetAsync(id, ct);
        if (product is not null)
        {
            await cache.SetAsync(key, JsonSerializer.SerializeToUtf8Bytes(product),
                new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10) }, ct);
        }
        return product;
    }
}
```

### HybridCache (.NET 9 — recommended)
```csharp
builder.Services.AddHybridCache(o =>
{
    o.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),       // L2
        LocalCacheExpiration = TimeSpan.FromMinutes(2) // L1
    };
});

public sealed class CatalogService(HybridCache cache, ICatalogRepo repo)
{
    public ValueTask<Product?> GetAsync(int id, CancellationToken ct) =>
        cache.GetOrCreateAsync(
            key: $"product:{id}",
            factory: async ct2 => await repo.GetAsync(id, ct2),
            tags: new[] { "products", $"product:{id}" },
            cancellationToken: ct);

    public ValueTask InvalidateAsync(int id, CancellationToken ct) =>
        cache.RemoveAsync($"product:{id}", ct);

    public ValueTask InvalidateAllProductsAsync(CancellationToken ct) =>
        cache.RemoveByTagAsync("products", ct);
}
```

### Output cache (.NET 7+)
```csharp
builder.Services.AddOutputCache(o =>
{
    o.AddPolicy("Products", p => p.Expire(TimeSpan.FromMinutes(1)).Tag("products"));
});

app.UseOutputCache();
app.MapGet("/products", () => GetProducts()).CacheOutput("Products");

// Invalidate from anywhere
public static async Task EvictProducts(IOutputCacheStore store, CancellationToken ct) =>
    await store.EvictByTagAsync("products", ct);
```

### Response cache headers (CDN-friendly)
```csharp
app.MapGet("/static-things", (HttpContext ctx) =>
{
    ctx.Response.Headers.CacheControl = "public, max-age=3600";
    return Results.Ok(thing);
});

[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any)]
public IActionResult Index() => View();
```

### Stampede protection (manual)
```csharp
private static readonly SemaphoreSlim _lock = new(1, 1);

public async Task<T> GetAsync<T>(string key, Func<Task<T>> factory)
{
    if (_cache.TryGetValue<T>(key, out var v)) return v;
    await _lock.WaitAsync();
    try
    {
        if (_cache.TryGetValue<T>(key, out v)) return v;   // double-check
        v = await factory();
        _cache.Set(key, v, TimeSpan.FromMinutes(5));
        return v;
    }
    finally { _lock.Release(); }
}
```

(`HybridCache` does this automatically — prefer it.)

### Eviction callback
```csharp
cache.Set(key, value, new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5),
    PostEvictionCallbacks = { new PostEvictionCallbackRegistration
    {
        EvictionCallback = (k, v, reason, _) => _log.LogInformation("Evicted {Key} ({Reason})", k, reason)
    }}
});
```

### Redis pub/sub for cross-instance invalidation
```csharp
var subscriber = redis.GetSubscriber();
await subscriber.SubscribeAsync("invalidate", (_, key) => _cache.Remove((string)key!));

// On update:
await subscriber.PublishAsync("invalidate", $"user:{id}");
```

---

## Common Patterns

```csharp
// Pattern: cache-aside with negative caching (avoid retry storms on 404s)
public async Task<User?> GetAsync(int id)
{
    var key = $"user:{id}";
    if (_cache.TryGetValue<User?>(key, out var u)) return u;

    u = await _repo.GetAsync(id);
    _cache.Set(key, u,
        u is null ? TimeSpan.FromSeconds(30)   // short for misses
                  : TimeSpan.FromMinutes(5));
    return u;
}
```

```csharp
// Pattern: tag-based invalidation
await cache.GetOrCreateAsync(
    $"order:{id}",
    factory: ..., tags: new[] { "orders", $"user:{userId}:orders" });

await cache.RemoveByTagAsync($"user:{userId}:orders", ct);
```

```csharp
// Pattern: write-through (update DB and cache in one method)
public async Task UpdateAsync(User u)
{
    await _repo.UpdateAsync(u);
    _cache.Set($"user:{u.Id}", u, TimeSpan.FromMinutes(5));
}
```

```csharp
// Pattern: cache invalidation on save in DbContext
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    var dirtyKeys = ChangeTracker.Entries<User>().Select(e => $"user:{e.Entity.Id}").ToList();
    var result = await base.SaveChangesAsync(ct);
    foreach (var k in dirtyKeys) _cache.Remove(k);
    return result;
}
```

---

## Gotchas & Tips

- **Pick a TTL you can defend** — "infinite" caches drift; "5s" wastes effort. Tie TTL to staleness tolerance.
- **Cache the right shape** — store DTOs, not EF entities (no proxies, no tracking, smaller).
- **`IMemoryCache` is process-local** — across instances you get inconsistent reads. Use `HybridCache` or a distributed cache.
- **Set `SizeLimit`** on `IMemoryCache`, otherwise unbounded growth. Each entry must declare `Size`.
- **Stampede on cold start** — first miss after deploy gets hammered. `HybridCache` and `LazyCache` handle it; manual locking otherwise.
- **Negative caching** prevents thundering herds on common 404s, but use a short TTL.
- **Don't cache personalized data with shared keys** — leak risk. Include user ID in the key.
- **Serializer cost** — JSON to Redis is the default but slow for large payloads. MessagePack/protobuf is faster; raw bytes for binaries.
- **Distributed cache TTL is the source of truth** — don't fight it from the application side.
- **Cache key conventions** — use a stable prefix (`shop:`), include version (`shop:v2:user:1`) so a deploy can wipe stale shapes.
- **Output cache is opt-in by default** — you must call `app.UseOutputCache()` and decorate endpoints. Don't expect ASP.NET to cache "free".
- **Don't cache `IDisposable` resources** — connections, streams. Cache pure data.

---

## See Also

- [[06 - Performance Optimization]]
- [[16 - Entity Framework Core]]
- [[18 - Logging]]
- [[12 - Background Services]]
