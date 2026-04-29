---
tags: [dotnet, beginner, csharp, linq]
aliases: [LINQ, Select Where]
level: Beginner
---

# LINQ Basics

> **One-liner**: LINQ (Language-INtegrated Query) is a fluent SQL-like API on `IEnumerable<T>` for filtering, projecting, ordering, grouping, and aggregating — works on collections, databases (EF Core), XML, and any sequence.

---

## Quick Reference

| Operator | Returns | Purpose |
|----------|---------|---------|
| `Where(p)` | `IEnumerable<T>` | Filter by predicate |
| `Select(f)` | `IEnumerable<U>` | Transform / project |
| `SelectMany(f)` | `IEnumerable<U>` | Flatten nested sequences |
| `OrderBy(k)` / `OrderByDescending` | `IOrderedEnumerable<T>` | Sort |
| `ThenBy(k)` | chained sort | Secondary sort key |
| `GroupBy(k)` | `IEnumerable<IGrouping<K,T>>` | Group by key |
| `Distinct()` | `IEnumerable<T>` | Remove duplicates |
| `Take(n)` / `Skip(n)` | `IEnumerable<T>` | Paging |
| `First()` / `FirstOrDefault()` | `T` / `T?` | First match (or default) |
| `Single()` / `SingleOrDefault()` | `T` / `T?` | Exactly one (throws if multiple) |
| `Last()` / `LastOrDefault()` | `T` / `T?` | Last match |
| `Any(p?)` / `All(p)` | `bool` | Existential / universal |
| `Count(p?)` / `LongCount` | `int` / `long` | How many |
| `Sum/Min/Max/Average` | scalar | Aggregations |
| `Aggregate(seed, f)` | scalar | Custom fold |
| `Contains(x)` | `bool` | Membership |
| `ToList() / ToArray() / ToDictionary(k)` | concrete collection | Materialize |

---

## Core Concept

LINQ operators are **extension methods** on `IEnumerable<T>` that return new `IEnumerable<T>` — they compose. The query is **lazy**: nothing executes until you enumerate (`foreach`, `ToList()`, `First()`, etc.). This is called **deferred execution**.

There are two syntaxes — **method syntax** (chained calls) and **query syntax** (SQL-like keywords). They're equivalent; method syntax is more flexible and far more common in modern code.

LINQ in EF Core looks identical but **runs on the database** — see [[16 - Entity Framework Core]].

---

## Syntax & API

### Method syntax
```csharp
var people = new[]
{
    new { Name = "Alice", Age = 30, City = "NYC" },
    new { Name = "Bob",   Age = 25, City = "LA"  },
    new { Name = "Carol", Age = 35, City = "NYC" },
    new { Name = "Dave",  Age = 28, City = "LA"  },
};

// Filter + project + sort
var nycNames = people
    .Where(p => p.City == "NYC")
    .OrderBy(p => p.Age)
    .Select(p => p.Name)
    .ToList();
// → ["Alice", "Carol"]
```

### Query syntax
```csharp
var nycNames = (from p in people
                where p.City == "NYC"
                orderby p.Age
                select p.Name).ToList();
```

### First / Single / Default
```csharp
var first = people.First();                              // throws if empty
var firstOrNull = people.FirstOrDefault();               // null if empty
var alice = people.First(p => p.Name == "Alice");
var match = people.FirstOrDefault(p => p.Age > 100);     // null

var theNyc = people.Single(p => p.Name == "Alice");      // throws if 0 or >1
```

### Aggregations
```csharp
int total = people.Count();
int adults = people.Count(p => p.Age >= 18);
int sumAges = people.Sum(p => p.Age);
double avgAge = people.Average(p => p.Age);
int oldest = people.Max(p => p.Age);
var oldestPerson = people.MaxBy(p => p.Age);             // .NET 6+

bool anyTeen = people.Any(p => p.Age < 20);
bool allAdult = people.All(p => p.Age >= 18);
```

### GroupBy
```csharp
var byCity = people
    .GroupBy(p => p.City)
    .Select(g => new
    {
        City = g.Key,
        Count = g.Count(),
        AvgAge = g.Average(p => p.Age)
    });
// → [{NYC, 2, 32.5}, {LA, 2, 26.5}]
```

### Join
```csharp
var orders = new[]
{
    new { UserId = 1, Total = 100m },
    new { UserId = 2, Total = 50m  },
};
var users = new[]
{
    new { Id = 1, Name = "Alice" },
    new { Id = 2, Name = "Bob"   },
};

var joined = users.Join(
    orders,
    u => u.Id,
    o => o.UserId,
    (u, o) => new { u.Name, o.Total });
```

### Set operations
```csharp
var a = new[] { 1, 2, 3, 4 };
var b = new[] { 3, 4, 5, 6 };

var union     = a.Union(b);          // 1,2,3,4,5,6
var intersect = a.Intersect(b);      // 3,4
var except    = a.Except(b);         // 1,2
var distinct  = new[] {1,1,2,3}.Distinct(); // 1,2,3
```

### Paging
```csharp
var page2 = items.OrderBy(x => x.Id)
                 .Skip(10)
                 .Take(10)
                 .ToList();
```

### Materializing
```csharp
List<int> list  = source.ToList();
int[]     arr   = source.ToArray();
HashSet<int> set = source.ToHashSet();
Dictionary<int, string> map = users.ToDictionary(u => u.Id, u => u.Name);
ILookup<string, User> byCity = users.ToLookup(u => u.City);
```

---

## Common Patterns

```csharp
// Pattern: top N per group
var top3PerCity = people
    .GroupBy(p => p.City)
    .SelectMany(g => g.OrderByDescending(p => p.Age).Take(3));
```

```csharp
// Pattern: word count
var counts = text
    .Split(' ', StringSplitOptions.RemoveEmptyEntries)
    .GroupBy(w => w.ToLowerInvariant())
    .ToDictionary(g => g.Key, g => g.Count());
```

```csharp
// Pattern: chunk a sequence (.NET 6+)
foreach (var batch in source.Chunk(100))
{
    await ProcessBatchAsync(batch);   // 100-item batches
}
```

---

## Gotchas & Tips

- **Deferred execution = surprise side effects** — calling `.Select(...)` on `_db.Users` doesn't run yet. The query runs on first enumeration. Materialize with `.ToList()` if you need to iterate twice.
- **Multiple enumeration** — calling `.First()` then iterating again re-runs the entire query (and may re-hit the DB). Cache to a list if needed.
- **`First()` vs `FirstOrDefault()`** — `First` throws on empty, `FirstOrDefault` returns `default(T)`. Pick based on whether absence is exceptional.
- **`Single` is stricter** — exactly one. Use it when uniqueness is invariant; otherwise `First`.
- **`Where` then `Select` is faster than `Select` then `Where`** in some cases — push filters down the chain.
- **LINQ on `IQueryable` (EF Core) translates to SQL** — not all C# expressions are supported. See [[16 - Entity Framework Core]].
- **`Count() > 0` ≈ `Any()`** but `Any()` is preferred — short-circuits, doesn't enumerate the whole sequence.

---

## See Also

- [[06 - Collections]]
- [[05 - LINQ Advanced]]
- [[16 - Entity Framework Core]]
- [[04 - Lambda and Functional]]
