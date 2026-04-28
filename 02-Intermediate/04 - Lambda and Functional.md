---
tags: [dotnet, intermediate, csharp]
aliases: [Lambda, Closures, Expression Trees]
level: Intermediate
---

# Lambda and Functional

> **One-liner**: Lambdas are inline functions that close over their lexical scope; expression trees represent code as data; functional patterns (immutability, pure functions, higher-order methods) are first-class in modern C#.

---

## Quick Reference

| Form | Example |
|------|---------|
| Zero-arg | `() => 42` |
| Single-arg | `x => x * 2` |
| Multi-arg | `(a, b) => a + b` |
| Typed args | `(int a, int b) => a + b` |
| Block body | `x => { Log(x); return x * 2; }` |
| Async | `async x => await Get(x)` |
| Static (no capture) | `static x => x * 2` |
| Discard param | `_ => 42` |
| Default value (C# 12+) | `(int x = 5) => x * 2` |
| `params` (C# 13+) | `(params int[] xs) => xs.Sum()` |
| Expression tree | `Expression<Func<int,int>> e = x => x + 1;` |

---

## Core Concept

A **lambda** is an anonymous function. The compiler turns it into a hidden method (and a hidden class if it captures variables — that's a **closure**).

A **closure** captures variables by **reference**, not value — the lambda sees the *current* value of the variable, not the value at lambda-creation time. This causes the classic "loop variable capture" bug.

**Expression trees** (`Expression<Func<...>>`) represent the lambda as a tree of nodes (parameter, binary-op, member-access, ...) instead of compiled code. EF Core, Moq, and many libraries inspect them to translate C# to SQL/proxies/etc.

**Pure functions** (no side effects, deterministic) and **immutable data** (records) are functional cornerstones — they make code easier to test and reason about.

---

## Syntax & API

### Lambda forms
```csharp
Func<int, int> square = x => x * x;
Func<int, int, int> add = (a, b) => a + b;
Func<int> rand = () => Random.Shared.Next();

// Block body when more than one statement
Func<int, int> doubleAndLog = x =>
{
    Console.WriteLine($"input: {x}");
    return x * 2;
};

// Async lambda
Func<string, Task<string>> fetch = async url =>
    await new HttpClient().GetStringAsync(url);
```

### Closures
```csharp
int factor = 3;
Func<int, int> scale = x => x * factor;

Console.WriteLine(scale(10));     // 30

factor = 5;                        // captured by reference!
Console.WriteLine(scale(10));     // 50 — sees new value
```

### Loop capture pitfall (and fix)
```csharp
// ❌ All lambdas print the SAME final value of i (foreach in old C# only)
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
foreach (var a in actions) a();      // 3 3 3

// ✅ Capture per-iteration
for (int i = 0; i < 3; i++)
{
    int captured = i;
    actions.Add(() => Console.WriteLine(captured));
}
// 0 1 2

// ✅ foreach in modern C# already captures per-iteration
foreach (var x in list) actions.Add(() => Console.WriteLine(x));   // safe
```

### Static lambdas (no capture)
```csharp
// Compiler enforces no captures — zero allocation
Func<int, int> pure = static x => x * 2;

int multiplier = 3;
// Func<int, int> bad = static x => x * multiplier;  // ❌ compile error
```

### Expression tree
```csharp
Expression<Func<int, bool>> isPositive = n => n > 0;

// You can inspect or compile the expression
Console.WriteLine(isPositive);                     // n => (n > 0)

Func<int, bool> compiled = isPositive.Compile();
Console.WriteLine(compiled(5));                    // true

// EF Core: this expression gets translated to SQL "WHERE n > 0"
var positive = db.Numbers.Where(isPositive);
```

### Higher-order functions
```csharp
// Function that returns a function
Func<int, Func<int, int>> adder = x => y => x + y;

var add5 = adder(5);
Console.WriteLine(add5(3));        // 8

// Function that takes a function
public T Retry<T>(Func<T> action, int maxAttempts = 3)
{
    for (int i = 0; i < maxAttempts; i++)
    {
        try { return action(); }
        catch when (i < maxAttempts - 1) { Thread.Sleep(100); }
    }
    return action();    // last try lets exception propagate
}
```

---

## Common Patterns

```csharp
// Pattern: pipeline via local functions and LINQ
public IEnumerable<string> Process(IEnumerable<string> input) =>
    input
        .Where(s => !string.IsNullOrWhiteSpace(s))
        .Select(s => s.Trim().ToLowerInvariant())
        .Where(s => s.Length > 2)
        .Distinct();
```

```csharp
// Pattern: immutable records + pure transforms
public record Order(int Id, decimal Total, OrderStatus Status);

public Order MarkPaid(Order o) => o with { Status = OrderStatus.Paid };
public Order ApplyDiscount(Order o, decimal pct) =>
    o with { Total = o.Total * (1 - pct) };
```

```csharp
// Pattern: memoization via captured dictionary
public static Func<TIn, TOut> Memoize<TIn, TOut>(Func<TIn, TOut> fn) where TIn : notnull
{
    var cache = new Dictionary<TIn, TOut>();
    return input =>
    {
        if (!cache.TryGetValue(input, out var output))
            cache[input] = output = fn(input);
        return output;
    };
}
```

---

## Gotchas & Tips

- **Capturing allocates** — every closure with captured variables creates a hidden class instance. Hot paths should avoid captures or use `static` lambdas.
- **`Expression<T>` cannot have a block body** — only single-expression lambdas. `Expression<Func<int,int>> e = x => { return x + 1; };` won't compile.
- **`Compile()` on an expression is expensive** (~ms) — cache the result.
- **Closures over `for`-loop variables capture the same slot** — the modern foreach (since C# 5) captures per-iteration, but `for` does not.
- **Lambdas hide complexity in stack traces** — the synthesized method names like `<>__DisplayClass1_0` make debugging harder. Prefer named methods for non-trivial logic.
- **Don't return lambdas from public APIs** unless you really mean it — they're harder to test, mock, and document than interfaces.

---

## See Also

- [[03 - Delegates and Events]]
- [[10 - LINQ Basics]]
- [[05 - LINQ Advanced]]
- [[06 - Performance Optimization]]
