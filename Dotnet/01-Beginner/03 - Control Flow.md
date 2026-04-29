---
tags: [dotnet, beginner, csharp]
aliases: [Conditionals, Loops, Pattern Matching]
level: Beginner
---

# Control Flow

> **One-liner**: C# offers familiar `if`/`switch`/loop constructs plus modern **pattern matching** that lets `switch` and `is` deconstruct types and shapes.

---

## Quick Reference

| Construct | Use for |
|-----------|---------|
| `if` / `else if` / `else` | Standard branching |
| `switch` (statement) | Multi-way branch on a value |
| `switch` (expression) | Branch returning a value |
| `for` | Counted iteration |
| `foreach` | Iterate any `IEnumerable<T>` |
| `while` / `do…while` | Condition-based loop |
| `break` / `continue` | Exit / skip iteration |
| `return` | Exit method |
| `goto` | (rare) jump to label |
| `is` / `is not` | Type/pattern test |
| `?:` | Ternary expression |

---

## Core Concept

Control flow in C# is mostly conventional, but **pattern matching** (since C# 7+, expanded through C# 11) is a superpower: `switch` can match on type, property values, tuple shape, list shape, and ranges — all in one expression that returns a value.

Modern C# strongly prefers **expressions over statements** because expressions compose: a `switch` expression can sit inside an interpolated string, a LINQ projection, or a constructor argument.

---

## Syntax & API

### if / else
```csharp
if (age >= 18 && hasLicense)
    Console.WriteLine("Can drive");
else if (age >= 16)
    Console.WriteLine("Permit");
else
    Console.WriteLine("Too young");
```

### switch statement
```csharp
switch (day)
{
    case DayOfWeek.Saturday:
    case DayOfWeek.Sunday:
        Console.WriteLine("Weekend");
        break;
    case DayOfWeek.Friday:
        Console.WriteLine("Almost weekend");
        break;
    default:
        Console.WriteLine("Weekday");
        break;
}
```

### switch expression (preferred)
```csharp
string label = day switch
{
    DayOfWeek.Saturday or DayOfWeek.Sunday => "Weekend",
    DayOfWeek.Friday                       => "Almost weekend",
    _                                      => "Weekday"
};
```

### Pattern matching
```csharp
// Type pattern + property pattern
string Describe(object o) => o switch
{
    int n when n < 0       => "negative int",
    int 0                  => "zero",
    int n                  => $"int {n}",
    string { Length: 0 }   => "empty string",
    string s               => $"string of {s.Length}",
    null                   => "null",
    _                      => "unknown"
};

// is patterns
if (shape is Circle { Radius: > 0 } c)
    Console.WriteLine($"Circle area: {Math.PI * c.Radius * c.Radius}");
```

### Loops
```csharp
for (int i = 0; i < 10; i++) { /* ... */ }

foreach (var item in collection) { /* ... */ }

while (queue.Count > 0) { /* ... */ }

do { /* ... */ } while (retry && attempts < 3);

// Range/index (C# 8+)
int[] nums = { 1, 2, 3, 4, 5 };
foreach (var n in nums[1..4]) Console.WriteLine(n); // 2, 3, 4
```

---

## Common Patterns

```csharp
// Pattern: replace if/else chains with switch expression
decimal CalculateFee(Account account) => account.Tier switch
{
    "Premium"  => 0m,
    "Standard" => 2.50m,
    "Free"     => 5.00m,
    _          => throw new InvalidOperationException("Unknown tier")
};
```

```csharp
// Pattern: deconstruct tuples in switch
(string, int) input = GetData();
var msg = input switch
{
    ("admin", _)         => "Admin user",
    (_, > 100)           => "Power user",
    (var name, var age)  => $"{name}, age {age}"
};
```

```csharp
// Pattern: early return reduces nesting
public bool TryProcess(Order? order)
{
    if (order is null) return false;
    if (order.Items.Count == 0) return false;
    if (order.Total <= 0) return false;

    // happy path here, no nesting
    Process(order);
    return true;
}
```

---

## Gotchas & Tips

- **switch expression must be exhaustive** or include `_ =>` — non-exhaustive patterns produce a warning.
- **Fall-through is illegal** in `switch` statements (unlike C/C++). Use stacked `case` labels for OR.
- **Don't use `goto`** unless implementing a state machine — almost always a code smell.
- **Pattern matching doesn't replace polymorphism** — for behavior dispatch, prefer virtual methods. Use patterns for data shape inspection.
- **`foreach` on a null collection throws** — guard or use `?? Enumerable.Empty<T>()`.

---

## See Also

- [[02 - CSharp Basics]]
- [[05 - OOP Fundamentals]]
- [[10 - LINQ Basics]]
