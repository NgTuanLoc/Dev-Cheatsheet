---
tags: [dotnet, beginner, csharp]
aliases: [Functions, Method Parameters]
level: Beginner
---

# Methods

> **One-liner**: Methods are named, parameterized blocks of code; C# adds optional/named arguments, `ref`/`out`/`in` parameter modes, expression-bodied syntax, and local functions.

---

## Quick Reference

| Feature | Syntax |
|---------|--------|
| Standard method | `int Add(int a, int b) { return a + b; }` |
| Expression-bodied | `int Add(int a, int b) => a + b;` |
| Optional parameter | `void Log(string msg, int level = 0)` |
| Named arguments | `Log(msg: "hi", level: 2)` |
| `params` array | `int Sum(params int[] nums)` |
| `out` parameter | `bool TryParse(string s, out int v)` |
| `ref` parameter | `void Swap(ref int a, ref int b)` |
| `in` parameter | `void Read(in BigStruct s)` (readonly ref) |
| Tuple return | `(int min, int max) MinMax(...)` |
| Local function | `void Outer() { int Helper() => 1; }` |
| Static method | `static int Square(int x) => x * x;` |
| Async method | `async Task<int> FetchAsync()` |

---

## Core Concept

A method is a unit of behavior. In C# methods always live inside a type (class, struct, record, interface). The signature is `[modifiers] returnType Name(parameters)`.

Parameters can be passed:
- **By value** (default) — a copy is passed; mutations don't propagate
- **By reference (`ref`)** — caller's variable can be read AND written
- **Out-only (`out`)** — must be assigned before the method returns
- **In (`in`)** — readonly reference, avoids copying large structs

Modern C# lets you write expression-bodied members for one-liner methods, return tuples instead of needing `out` for multi-value returns, and define **local functions** scoped inside a method.

---

## Syntax & API

### Basic method
```csharp
public int Add(int a, int b)
{
    return a + b;
}

// Expression-bodied (one-liner)
public int Multiply(int a, int b) => a * b;
```

### Optional + named arguments
```csharp
public void SendEmail(string to, string subject, bool html = false, int retries = 3)
{
    /* ... */
}

SendEmail("a@b.com", "Hi");                              // defaults
SendEmail("a@b.com", "Hi", html: true);                  // named
SendEmail(to: "a@b.com", subject: "Hi", retries: 5);     // reordered by name
```

### params (variable args)
```csharp
public int Sum(params int[] numbers)
{
    int total = 0;
    foreach (var n in numbers) total += n;
    return total;
}

Sum(1, 2, 3);          // implicit array
Sum(1, 2, 3, 4, 5);
Sum(new[] { 1, 2 });   // explicit array
```

### out / ref / in
```csharp
// out — caller doesn't need to initialize
public bool TryDivide(int a, int b, out int result)
{
    if (b == 0) { result = 0; return false; }
    result = a / b;
    return true;
}

if (TryDivide(10, 3, out int q)) Console.WriteLine(q);

// ref — read AND write the caller's variable
public void Increment(ref int x) => x++;

int n = 5;
Increment(ref n);  // n is now 6

// in — readonly reference, perf optimization for large structs
public double Length(in Vector3 v) => Math.Sqrt(v.X * v.X + v.Y * v.Y + v.Z * v.Z);
```

### Multiple return values via tuple
```csharp
public (int min, int max) MinMax(int[] values)
{
    return (values.Min(), values.Max());
}

var (lo, hi) = MinMax(new[] { 3, 1, 4, 1, 5 });
```

### Local function
```csharp
public int ProcessOrder(Order order)
{
    int Tax(int amount) => amount * 10 / 100;        // private helper
    return order.Subtotal + Tax(order.Subtotal);
}
```

### Method overloading
```csharp
public void Print(int x) => Console.WriteLine($"int: {x}");
public void Print(string s) => Console.WriteLine($"str: {s}");
public void Print(int x, int y) => Console.WriteLine($"({x},{y})");
```

---

## Common Patterns

```csharp
// Pattern: TryXxx instead of throwing
public bool TryGetUser(int id, out User? user)
{
    user = _users.FirstOrDefault(u => u.Id == id);
    return user is not null;
}
```

```csharp
// Pattern: tuple return for "result + status"
public (bool success, string message) Validate(Order o)
{
    if (o.Items.Count == 0) return (false, "No items");
    if (o.Total < 0)        return (false, "Negative total");
    return (true, "OK");
}
```

---

## Gotchas & Tips

- **Don't overload with `params`** when a same-arity overload exists — the compiler may pick the wrong one.
- **`ref` and `out` aren't interchangeable** — `out` says "I will assign", `ref` says "I might read AND assign".
- **Default parameter values are baked into the caller** at compile time. If you change `int retries = 3` to `5`, callers must recompile or they keep using `3`.
- **Local functions are zero-allocation** when they don't capture variables — preferred over lambdas for inline helpers.
- **Method overload resolution gets confusing fast** — keep overloads to a minimum, or use distinct method names.

---

## See Also

- [[03 - Control Flow]]
- [[05 - OOP Fundamentals]]
- [[03 - Delegates and Events]]
- [[04 - Lambda and Functional]]
