---
tags: [dotnet, beginner, csharp]
aliases: [Try Catch, Exceptions]
level: Beginner
---

# Exception Handling

> **One-liner**: Exceptions interrupt the normal flow when something goes wrong; `try`/`catch`/`finally` recovers, `using` ensures cleanup, and you should throw **specific** types — never plain `Exception`.

---

## Quick Reference

| Construct | Purpose |
|-----------|---------|
| `try { } catch { }` | Catch any exception |
| `catch (FooException ex)` | Catch a specific type |
| `catch (Exception ex) when (cond)` | Filter — only catch if `cond` true |
| `finally` | Always runs (cleanup) |
| `throw;` | Rethrow preserving stack trace |
| `throw ex;` | Rethrow **losing** stack trace (avoid) |
| `throw new XException(...)` | Throw a new exception |
| `using` | Auto-dispose at scope end (sugar for try/finally) |

| Common exception | Meaning |
|------------------|---------|
| `ArgumentNullException` | Argument was null |
| `ArgumentException` | Argument invalid |
| `ArgumentOutOfRangeException` | Index/value outside range |
| `InvalidOperationException` | Object state invalid for operation |
| `NotSupportedException` | Operation not supported here |
| `NotImplementedException` | Stub — temporary marker |
| `KeyNotFoundException` | Dictionary key missing |
| `IndexOutOfRangeException` | Array/list index invalid |
| `NullReferenceException` | Dereferenced null (almost always a bug) |
| `OperationCanceledException` | `CancellationToken` cancelled |
| `TimeoutException` | Operation timed out |
| `IOException` / `FileNotFoundException` | File system errors |
| `HttpRequestException` | HTTP call failed |

---

## Core Concept

An **exception** is an object describing an error condition. When thrown, the runtime walks up the call stack looking for a matching `catch`. If none found, the process crashes.

Exceptions are **expensive** — capturing the stack trace costs microseconds. Use them for **truly exceptional** conditions, not for normal control flow (e.g. don't throw to signal "user not found" — return a nullable or a `Result` type).

`finally` runs whether or not an exception occurred — perfect for releasing resources. `using` is syntactic sugar for `try/finally + Dispose()` (see [[10 - IDisposable and Resource Mgmt]]).

---

## Syntax & API

### Basic try / catch / finally
```csharp
try
{
    var json = File.ReadAllText("config.json");
    var config = JsonSerializer.Deserialize<Config>(json);
}
catch (FileNotFoundException ex)
{
    Console.Error.WriteLine($"Config missing: {ex.FileName}");
}
catch (JsonException ex)
{
    Console.Error.WriteLine($"Bad JSON: {ex.Message}");
}
catch (Exception ex)              // last resort — log and rethrow
{
    Logger.Error(ex, "Unexpected");
    throw;                         // preserve stack
}
finally
{
    // always runs
    CleanupTempFiles();
}
```

### Exception filters
```csharp
try { /* ... */ }
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // only catch 404s
}
catch (HttpRequestException ex) when (ex.StatusCode >= HttpStatusCode.InternalServerError)
{
    // only catch 5xx — let 4xx propagate
}
```

### Throwing
```csharp
public void Process(Order order)
{
    ArgumentNullException.ThrowIfNull(order);             // .NET 6+

    if (order.Items.Count == 0)
        throw new ArgumentException("Order must have items", nameof(order));

    if (order.Total < 0)
        throw new InvalidOperationException("Negative total");
}
```

### Custom exception
```csharp
public class InsufficientFundsException : Exception
{
    public decimal Balance { get; }
    public decimal Requested { get; }

    public InsufficientFundsException(decimal balance, decimal requested)
        : base($"Balance {balance} < requested {requested}")
    {
        Balance = balance;
        Requested = requested;
    }
}
```

### Rethrowing correctly
```csharp
try { /* ... */ }
catch (Exception ex)
{
    Logger.Error(ex);
    throw;                  // ✅ preserves original stack trace
    // throw ex;            // ❌ resets stack trace to here
}

// To wrap with context, use inner exception
try { /* ... */ }
catch (Exception ex)
{
    throw new ApplicationException("While processing order", ex);
}
```

### using — guaranteed disposal
```csharp
// Statement form
using (var stream = File.OpenRead("data.bin"))
{
    // ... read ...
}   // Dispose() called here, even on exception

// Declaration form (C# 8+)
public void Process()
{
    using var stream = File.OpenRead("data.bin");
    // ... read ...
}   // disposed at end of method
```

---

## Common Patterns

```csharp
// Pattern: validate inputs at boundary
public void Withdraw(decimal amount)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(amount);
    if (amount > Balance)
        throw new InsufficientFundsException(Balance, amount);
    Balance -= amount;
}
```

```csharp
// Pattern: don't catch what you can't handle
try
{
    return await client.GetAsync(url);
}
catch (HttpRequestException) when (retryCount < 3)
{
    retryCount++;
    await Task.Delay(1000);
    return await Retry();
}
// let other exceptions propagate
```

```csharp
// Pattern: global handler for ASP.NET Core (in Program.cs)
app.UseExceptionHandler(exceptionHandlerApp =>
{
    exceptionHandlerApp.Run(async context =>
    {
        var ex = context.Features.Get<IExceptionHandlerPathFeature>()?.Error;
        await Results.Problem(detail: ex?.Message).ExecuteAsync(context);
    });
});
```

---

## Gotchas & Tips

- **Never `catch (Exception)` and swallow** — at minimum log it. Swallowing hides bugs.
- **Don't use exceptions for control flow** — they're 100–1000× slower than normal returns. Prefer `bool TryX(...)` or `Result<T>`.
- **`throw;` ≠ `throw ex;`** — only `throw;` preserves the original stack trace.
- **`async` methods capture exceptions in the returned `Task`** — they fire when you `await`. `void async` methods crash the process — never use except in event handlers.
- **`finally` runs even on `return`** — but if `finally` throws, it overwrites the original exception. Keep `finally` blocks simple.
- **`AggregateException` wraps multiple** errors from `Task.WhenAll` and parallel ops — call `.Flatten()` to inspect.

---

## See Also

- [[10 - IDisposable and Resource Mgmt]]
- [[06 - Async and Await]]
- [[18 - Logging]]
