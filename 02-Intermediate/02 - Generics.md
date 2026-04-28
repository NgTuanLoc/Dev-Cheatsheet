---
tags: [dotnet, intermediate, csharp]
aliases: [Generic Types, Constraints]
level: Intermediate
---

# Generics

> **One-liner**: Generics let types and methods be parameterized over types — write once, get type-safe `List<int>`, `List<User>`, etc., with **no boxing** and full IntelliSense.

---

## Quick Reference

| Feature | Syntax |
|---------|--------|
| Generic class | `class Box<T> { T Value; }` |
| Generic method | `T First<T>(IEnumerable<T> xs)` |
| Multiple type params | `class Map<K, V>` |
| Constraint: must be class | `where T : class` |
| Constraint: must be struct | `where T : struct` |
| Constraint: parameterless ctor | `where T : new()` |
| Constraint: must inherit | `where T : Animal` |
| Constraint: must implement | `where T : IComparable<T>` |
| Constraint: notnull | `where T : notnull` |
| Constraint: unmanaged | `where T : unmanaged` |
| Combined | `where T : class, IDisposable, new()` |
| Default value | `default(T)` or `default` |
| Open generic | `typeof(List<>)` |
| Closed generic | `typeof(List<int>)` |

---

## Core Concept

Generics are resolved at compile time for value types (each `List<int>`, `List<double>` becomes a distinct JIT'd specialization) and shared at runtime for reference types (one `List<object>`-style implementation, with type checks). Either way, you get **type safety** and avoid casts.

**Constraints** narrow what types are allowed, unlocking specific operations: `where T : IComparable<T>` lets you call `.CompareTo()`. Without a constraint, all you can do with `T` is treat it as `object`.

**Variance** — for interfaces and delegates, `out` (covariant) means you can use a subtype where a supertype is expected (`IEnumerable<Dog>` → `IEnumerable<Animal>`); `in` (contravariant) is the reverse.

---

## Syntax & API

### Generic class
```csharp
public class Cache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _data = new();

    public void Set(TKey key, TValue value) => _data[key] = value;

    public TValue? Get(TKey key) =>
        _data.TryGetValue(key, out var v) ? v : default;
}

var cache = new Cache<string, User>();
cache.Set("alice", alice);
```

### Generic method
```csharp
public static T MaxOf<T>(T a, T b) where T : IComparable<T>
{
    return a.CompareTo(b) > 0 ? a : b;
}

int m = MaxOf(3, 7);            // T inferred as int
string s = MaxOf("a", "z");
```

### Multiple constraints
```csharp
public T Create<T>() where T : class, new()
{
    return new T();
}

public void Log<T>(T value) where T : notnull, IFormattable
{
    Console.WriteLine(value.ToString("G", null));
}
```

### Generic interface
```csharp
public interface IRepository<T> where T : class, IEntity
{
    Task<T?> GetByIdAsync(int id);
    Task AddAsync(T entity);
    Task<IReadOnlyList<T>> ListAsync();
}

public class UserRepository : IRepository<User> { /* ... */ }
```

### Variance
```csharp
// out = covariant — IEnumerable<T> only produces T's
IEnumerable<Animal> animals = new List<Dog>();   // ✅

// in = contravariant — Action<T> only consumes T's
Action<Dog> dogAction = (Dog d) => Console.WriteLine(d.Name);
Action<Puppy> puppyAction = dogAction;           // ✅ a Puppy IS-A Dog

// invariant (default) — must match exactly
List<Animal> list = new List<Dog>();             // ❌ compile error
```

### Custom variant interface
```csharp
public interface IProducer<out T>
{
    T Produce();              // T only on output
}

public interface IConsumer<in T>
{
    void Consume(T item);     // T only on input
}
```

### default for generic
```csharp
public T? FirstOrFallback<T>(IEnumerable<T> source)
{
    foreach (var x in source) return x;
    return default;            // null for class, 0/false/etc. for struct
}
```

---

## Common Patterns

```csharp
// Pattern: generic Result<T> instead of exceptions
public readonly record struct Result<T>(bool Success, T? Value, string? Error)
{
    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(string err) => new(false, default, err);
}

Result<User> r = Result<User>.Ok(user);
```

```csharp
// Pattern: generic factory with new() constraint
public class Pool<T> where T : new()
{
    private readonly Stack<T> _items = new();

    public T Rent()  => _items.Count > 0 ? _items.Pop() : new T();
    public void Return(T item) => _items.Push(item);
}
```

```csharp
// Pattern: type parameter as a marker (phantom types)
public readonly struct Id<TEntity>(int Value);

Id<User> userId = new(42);
Id<Order> orderId = new(99);
// userId == orderId   // ❌ different types, won't compile
```

---

## Gotchas & Tips

- **Reference-type generic methods share JIT code** — they don't get specialized like value types. Performance is identical to using `object`, but type safety is much better.
- **Value-type generics avoid boxing** — `List<int>` stores `int`s directly; `ArrayList` boxes each into `object`.
- **You cannot do arithmetic on `T`** without a constraint. Use `INumber<T>` (.NET 7+) for generic math.
- **`default(T)` on a non-nullable reference type returns `null`** — the compiler may warn. Use `T?` in the declaration.
- **`new T()` requires the `new()` constraint** AND a public parameterless constructor — costs a small reflection-style call.
- **Variance only works on interfaces and delegates**, not concrete classes. `List<Dog>` is never assignable to `List<Animal>`.

---

## See Also

- [[05 - OOP Fundamentals]]
- [[01 - OOP Advanced]]
- [[06 - Collections]]
