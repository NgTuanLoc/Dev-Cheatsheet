---
tags: [dotnet, intermediate, csharp, linq]
aliases: [Advanced LINQ, Joins, Aggregations]
level: Intermediate
---

# LINQ Advanced

> **One-liner**: Beyond the basics — joins, custom aggregations, deferred-vs-immediate execution, `IQueryable` vs `IEnumerable`, query syntax for complex shapes, and writing your own LINQ-style operators.

---

## Quick Reference

| Concept | Notes |
|---------|-------|
| `IEnumerable<T>` | In-memory, executes via `MoveNext` (LINQ to Objects) |
| `IQueryable<T>` | Expression tree, translated by provider (LINQ to SQL/EF/...) |
| Deferred | `Where`, `Select`, `OrderBy`, `GroupBy`, etc. |
| Immediate | `ToList`, `Count`, `First`, `Sum`, `Aggregate` |
| Streaming | `Where`, `Select` (yield as you go) |
| Buffering | `OrderBy`, `GroupBy`, `Reverse` (must read all first) |

---

## Core Concept

LINQ has two worlds:

1. **LINQ to Objects** (`IEnumerable<T>`) — operators are extension methods on `Enumerable`, run in process, lambdas execute as compiled delegates.
2. **LINQ to Providers** (`IQueryable<T>`) — operators are extensions on `Queryable`, lambdas captured as `Expression<Func<...>>`, the provider (EF Core, MongoDB driver, ...) walks the tree and emits a backend query.

Mixing them: once you call something the provider can't translate (e.g. `.AsEnumerable()`), the rest runs in memory. Be deliberate about this — it's a frequent source of N+1 problems.

**Deferred execution** means LINQ chains build a *plan*; execution happens on enumeration. **Streaming** operators (`Where`, `Select`) yield items lazily; **buffering** operators (`OrderBy`, `GroupBy`) need to consume the whole input.

---

## Syntax & API

### Joins
```csharp
var users = new[]
{
    new { Id = 1, Name = "Alice" },
    new { Id = 2, Name = "Bob"   },
};
var orders = new[]
{
    new { UserId = 1, Total = 100m },
    new { UserId = 1, Total = 50m  },
    new { UserId = 3, Total = 200m },     // orphan
};

// Inner join
var inner = users.Join(orders,
    u => u.Id,
    o => o.UserId,
    (u, o) => new { u.Name, o.Total });

// Group join (one-to-many)
var groupJoin = users.GroupJoin(orders,
    u => u.Id,
    o => o.UserId,
    (u, os) => new { u.Name, Orders = os.ToList() });

// Left join (via GroupJoin + SelectMany + DefaultIfEmpty)
var left = users.GroupJoin(orders,
    u => u.Id,
    o => o.UserId,
    (u, os) => new { u, os })
.SelectMany(x => x.os.DefaultIfEmpty(),
    (x, o) => new { x.u.Name, Total = o?.Total ?? 0m });
```

### Query syntax for complex shapes
```csharp
var result =
    from u in users
    join o in orders on u.Id equals o.UserId into userOrders
    from o in userOrders.DefaultIfEmpty()
    let category = o == null ? "none" : o.Total > 75 ? "big" : "small"
    where category != "none"
    group new { u.Name, o.Total } by category into g
    orderby g.Key
    select new { Category = g.Key, Sum = g.Sum(x => x.Total) };
```

### let — introduce a name
```csharp
var named =
    from p in people
    let fullName = $"{p.First} {p.Last}"
    let age = (DateTime.Now - p.DateOfBirth).TotalDays / 365
    where age >= 18
    select new { fullName, age };
```

### Aggregate
```csharp
// Custom fold
int product = new[] { 1, 2, 3, 4 }.Aggregate(1, (acc, x) => acc * x); // 24

// Concatenate strings
string joined = new[] { "a", "b", "c" }.Aggregate((acc, x) => acc + "," + x);

// With result selector
var stats = nums.Aggregate(
    seed: (Sum: 0, Count: 0),
    func: (acc, n) => (acc.Sum + n, acc.Count + 1),
    resultSelector: acc => acc.Count == 0 ? 0 : (double)acc.Sum / acc.Count);
```

### Zip — pairwise combine
```csharp
var names  = new[] { "Alice", "Bob", "Carol" };
var scores = new[] { 90, 85, 78 };

foreach (var (n, s) in names.Zip(scores))
    Console.WriteLine($"{n}: {s}");
```

### Window functions (manual)
```csharp
// Running total via Aggregate
var runningTotals = orders
    .OrderBy(o => o.Date)
    .Scan(0m, (acc, o) => acc + o.Total);   // hypothetical Scan

// MoreLINQ has Scan, Window, Batch, etc. — worth knowing about
```

### Custom operator
```csharp
public static class MyLinq
{
    public static IEnumerable<TSource> WhereNot<TSource>(
        this IEnumerable<TSource> source,
        Func<TSource, bool> predicate)
    {
        foreach (var item in source)
            if (!predicate(item))
                yield return item;
    }
}

var nonAdults = people.WhereNot(p => p.Age >= 18);
```

### IQueryable vs IEnumerable
```csharp
IQueryable<User> q = db.Users.Where(u => u.IsActive);   // not yet executed

// Bad: AsEnumerable forces in-memory from here
var bad = q.AsEnumerable()
           .Where(u => u.Email.EndsWith("@admin.com"))   // C# filter
           .ToList();                                    // pulls ALL active users

// Good: keeps it in SQL
var good = q.Where(u => u.Email.EndsWith("@admin.com"))
            .ToList();                                   // single SQL query
```

---

## Common Patterns

```csharp
// Pattern: top-N per group with row number
var top3PerCategory = products
    .GroupBy(p => p.Category)
    .SelectMany(g => g.OrderByDescending(p => p.Sales).Take(3));
```

```csharp
// Pattern: full-stats per group
var summary = orders
    .GroupBy(o => o.UserId)
    .Select(g => new
    {
        UserId   = g.Key,
        Count    = g.Count(),
        Total    = g.Sum(o => o.Total),
        Avg      = g.Average(o => o.Total),
        FirstAt  = g.Min(o => o.CreatedAt),
        LastAt   = g.Max(o => o.CreatedAt),
    });
```

```csharp
// Pattern: deduplicate by key
var uniqueByEmail = users
    .GroupBy(u => u.Email)
    .Select(g => g.First());

// Or .NET 6+ shortcut
var unique = users.DistinctBy(u => u.Email);
```

```csharp
// Pattern: chunked async processing
foreach (var batch in items.Chunk(100))
{
    await db.SaveBatchAsync(batch);
}
```

---

## Gotchas & Tips

- **Multiple enumeration**: `var q = src.Where(...);` then iterating twice runs the pipeline twice. Materialize if you need to.
- **`Count() == 0` triggers full enumeration** in some providers — `Any()` short-circuits.
- **`OrderBy` is stable** in LINQ to Objects, but **not guaranteed** in `IQueryable` (depends on the SQL `ORDER BY`).
- **Avoid `ToList` mid-chain** unless you need the boundary — every materialization costs memory.
- **`SelectMany` on EF Core can produce huge cartesian products** if relationships aren't well-defined — verify the SQL.
- **`GroupBy` on EF Core has provider-specific limits** — some shapes won't translate; the provider may throw or silently move work to the client.
- **`Aggregate` with no seed throws on empty source** — pass a seed to be safe.
- **`DistinctBy`/`MinBy`/`MaxBy`/`Chunk`** require .NET 6+. Older projects need MoreLINQ.

---

## See Also

- [[10 - LINQ Basics]]
- [[16 - Entity Framework Core]]
- [[04 - Lambda and Functional]]
- [[06 - Performance Optimization]]
