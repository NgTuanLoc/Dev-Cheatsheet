---
tags: [dotnet, intermediate, csharp]
aliases: [Delegate, Action, Func, Events]
level: Intermediate
---

# Delegates and Events

> **One-liner**: A **delegate** is a type-safe function pointer; **events** are delegates with restricted access (only the declaring class can raise them) — together they enable callbacks and the publish/subscribe pattern.

---

## Quick Reference

| Built-in | Signature |
|----------|-----------|
| `Action` | `void()` |
| `Action<T>` | `void(T)` |
| `Action<T1,T2,...,T16>` | up to 16 args, returns void |
| `Func<TResult>` | `TResult()` |
| `Func<T,TResult>` | `TResult(T)` |
| `Func<T1,...,T16,TResult>` | up to 16 args + return |
| `Predicate<T>` | `bool(T)` (legacy; prefer `Func<T,bool>`) |
| `Comparison<T>` | `int(T,T)` (for sort) |
| `EventHandler` | `void(object?, EventArgs)` |
| `EventHandler<TArgs>` | `void(object?, TArgs)` |

| Operation | Syntax |
|-----------|--------|
| Combine (multicast) | `d1 + d2` or `+=` |
| Remove | `d1 - d2` or `-=` |
| Invoke | `d()` or `d.Invoke()` |
| Null-safe invoke | `d?.Invoke()` |

---

## Core Concept

A **delegate type** declares a method signature. A delegate **instance** wraps one or more methods matching that signature, and you can call all of them with one `()`.

**Events** wrap a delegate and only allow the declaring class to invoke it — outside code can only subscribe (`+=`) or unsubscribe (`-=`). This is the C# implementation of the **observer pattern**.

In modern C# you almost never declare your own delegate types — `Action`/`Func` cover everything. Custom `delegate` declarations are mostly for backwards compatibility or expressive naming.

---

## Syntax & API

### Action / Func basics
```csharp
Action greet = () => Console.WriteLine("Hi!");
greet();                                          // "Hi!"

Action<string> greetName = name => Console.WriteLine($"Hi, {name}");
greetName("Alice");

Func<int, int, int> add = (a, b) => a + b;
int r = add(2, 3);                                // 5

Predicate<int> isEven = n => n % 2 == 0;
bool e = isEven(4);                               // true
```

### Method group conversion
```csharp
void PrintLine(string s) => Console.WriteLine(s);

Action<string> p = PrintLine;        // method group → delegate
p("Hello");
```

### Multicast (combine multiple)
```csharp
Action a = () => Console.Write("1 ");
Action b = () => Console.Write("2 ");

Action both = a + b;
both();                              // "1 2"

both -= a;                           // remove
both();                              // "2"
```

### Custom delegate type (rare)
```csharp
public delegate decimal Discount(decimal price);

Discount d = price => price * 0.9m;
decimal final = d(100m);             // 90
```

### Events
```csharp
public class Stock
{
    public string Symbol { get; }
    public decimal Price { get; private set; }

    // Event declaration — only Stock can raise
    public event EventHandler<decimal>? PriceChanged;

    public Stock(string symbol, decimal price) { Symbol = symbol; Price = price; }

    public void Update(decimal newPrice)
    {
        Price = newPrice;
        PriceChanged?.Invoke(this, newPrice);    // raise (null-safe)
    }
}

// Subscribers
var aapl = new Stock("AAPL", 150);
aapl.PriceChanged += (sender, price) =>
    Console.WriteLine($"{((Stock)sender!).Symbol} → {price}");
aapl.PriceChanged += LogToFile;

aapl.Update(155);                    // both handlers fire

aapl.PriceChanged -= LogToFile;      // unsubscribe

void LogToFile(object? s, decimal p) { /* ... */ }
```

### Custom EventArgs
```csharp
public class TradeEventArgs : EventArgs
{
    public required string Symbol { get; init; }
    public required decimal Price { get; init; }
    public required int Quantity { get; init; }
}

public class Broker
{
    public event EventHandler<TradeEventArgs>? TradeExecuted;

    public void Execute(string sym, decimal p, int q)
    {
        TradeExecuted?.Invoke(this, new TradeEventArgs
        {
            Symbol = sym, Price = p, Quantity = q
        });
    }
}
```

---

## Common Patterns

```csharp
// Pattern: callback parameter
public async Task ProcessAsync(IEnumerable<Item> items, Action<Item> onProgress)
{
    foreach (var item in items)
    {
        await DoWork(item);
        onProgress(item);
    }
}

await ProcessAsync(items, item => Console.WriteLine($"Done: {item.Id}"));
```

```csharp
// Pattern: strategy via Func<>
public decimal CalculateDiscount(Order order, Func<Order, decimal> strategy)
    => strategy(order);

decimal d = CalculateDiscount(order, o => o.Total > 100 ? 10 : 0);
```

```csharp
// Pattern: weak event subscription via lambda capture (mind leaks!)
publisher.Event += handler;     // strong reference — must unsubscribe
// see Memory Leaks note for details
```

---

## Gotchas & Tips

- **Lambdas that capture variables allocate** — the compiler creates a closure class. In hot paths, prefer non-capturing lambdas or static lambdas (`static () => 1`).
- **`event` ≠ `delegate`** — outside code can't `=` assign to an event (would clear all subscribers). Inside the class, you read it like a field.
- **Always `?.Invoke(...)`** when raising events — they're null until someone subscribes, and threads can race.
- **Events cause memory leaks** — if a publisher outlives a subscriber, the publisher's event keeps the subscriber alive. Always unsubscribe when done. See [[07 - Memory Leaks and Profiling]].
- **`async void` event handlers** are the only legitimate `async void` — but exceptions can't be caught by the publisher. Catch internally.
- **Multicast `Func<>` only returns the last result** — earlier results are discarded.
- **Don't `await` an event** — they can't return Tasks. For async pub/sub use channels (see [[09 - Channels and Pipelines]]) or libraries like `MediatR`.

---

## See Also

- [[04 - Methods]]
- [[04 - Lambda and Functional]]
- [[07 - Memory Leaks and Profiling]]
- [[01 - Design Patterns]]
