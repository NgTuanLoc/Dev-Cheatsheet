---
tags: [dotnet, beginner, csharp]
aliases: [C# Basics, Variables and Types]
level: Beginner
---

# CSharp Basics

> **One-liner**: C# is a strongly-typed, compiled, object-oriented language with type inference, nullable reference types, and modern features like records and pattern matching.

---

## Quick Reference

| Type | Size | Range / Use |
|------|------|-------------|
| `bool` | 1 byte | `true`/`false` |
| `byte` / `sbyte` | 1 byte | 0–255 / -128–127 |
| `short` / `ushort` | 2 bytes | ±32K |
| `int` / `uint` | 4 bytes | ±2.1B (default integer) |
| `long` / `ulong` | 8 bytes | ±9.2 quintillion |
| `float` | 4 bytes | ~7 digits, suffix `f` |
| `double` | 8 bytes | ~15 digits (default float) |
| `decimal` | 16 bytes | ~28 digits, suffix `m`, money |
| `char` | 2 bytes | Single Unicode char `'A'` |
| `string` | reference | Immutable Unicode text |
| `object` | reference | Base of all types |
| `var` | inferred | Compiler picks the type |
| `dynamic` | runtime | Type checked at runtime |

---

## Core Concept

C# variables have a **declared type** that cannot change. The compiler enforces correctness at compile time — `int x = "hello";` won't compile. This catches huge classes of bugs early.

Types fall into two camps:
- **Value types** (`int`, `bool`, `struct`, `enum`) — stored on the stack, copied by value
- **Reference types** (`class`, `string`, arrays) — stored on the heap, copied by reference

**Nullable reference types** (enabled by `<Nullable>enable</Nullable>`) make `null` opt-in: `string` can't be null, `string?` can. The compiler warns when you forget to check.

---

## Syntax & API

### Variables and constants
```csharp
int age = 25;                  // Explicit type
var name = "Alice";            // Inferred (compile-time)
const double Pi = 3.14159;     // Compile-time constant
readonly DateTime CreatedAt;   // Set once in constructor

string? maybeNull = null;      // Nullable reference type
int? nullableInt = null;       // Nullable value type (Nullable<int>)
```

### Type conversions
```csharp
int i = 42;
long l = i;                    // Implicit (widening)
int back = (int)l;             // Explicit cast (narrowing)

string s = "123";
int parsed = int.Parse(s);                    // Throws on failure
bool ok = int.TryParse(s, out int result);    // Safe — preferred
double d = Convert.ToDouble("3.14");
```

### Null handling operators
```csharp
string? name = GetName();

string safe = name ?? "default";       // Null-coalescing
name ??= "default";                    // Assign if null
int? len = name?.Length;               // Null-conditional
string nonNull = name!;                // Null-forgiving (assert non-null)
```

### String interpolation
```csharp
var greeting = $"Hello, {name}! You are {age} years old.";
var formatted = $"Price: {price:C2}";   // Currency, 2 decimals
var multiline = $"""
    Name: {name}
    Age:  {age}
    """;                                // Raw string literal (C# 11+)
```

---

## Common Patterns

```csharp
// Pattern: TryParse over Parse for user input
if (int.TryParse(userInput, out int age) && age >= 0)
{
    Console.WriteLine($"Valid age: {age}");
}
else
{
    Console.WriteLine("Invalid input");
}
```

```csharp
// Pattern: nullable-safe chain
string? city = user?.Address?.City ?? "Unknown";
```

---

## Gotchas & Tips

- **`var` is not dynamic** — the type is inferred at compile time and locked in. `var x = 5;` makes `x` an `int` forever.
- **`==` on strings compares values** (because `string` overrides it), but on most reference types compares references. Use `.Equals()` for clarity.
- **`decimal` for money, never `double`** — `double` has binary rounding errors (`0.1 + 0.2 != 0.3`).
- **Nullable warnings ≠ errors** — they're warnings by default. Treat them as errors in CI: `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.
- **`string` is immutable** — every modification creates a new instance. Use `StringBuilder` for heavy concatenation (see [[07 - Strings]]).

---

## See Also

- [[01 - Dotnet Overview]]
- [[03 - Control Flow]]
- [[05 - OOP Fundamentals]]
- [[07 - Strings]]
