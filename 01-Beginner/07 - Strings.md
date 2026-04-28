---
tags: [dotnet, beginner, csharp]
aliases: [String, StringBuilder, Interpolation]
level: Beginner
---

# Strings

> **One-liner**: `string` in .NET is an **immutable** UTF-16 sequence; use interpolation for formatting, `StringBuilder` for heavy concatenation, and `Span<char>`/`ReadOnlySpan<char>` for zero-allocation parsing.

---

## Quick Reference

| Operation | Syntax |
|-----------|--------|
| Literal | `"hello"` |
| Verbatim (no escapes) | `@"C:\path"` |
| Interpolated | `$"Hello {name}"` |
| Raw (C# 11+) | `"""multi-line"""` |
| Length | `s.Length` |
| Concat | `a + b` or `string.Concat(...)` |
| Format | `string.Format("{0}", x)` |
| Compare ordinal | `a == b` or `string.Equals(a,b,StringComparison.Ordinal)` |
| Compare ignoring case | `string.Equals(a,b,StringComparison.OrdinalIgnoreCase)` |
| Split | `s.Split(',')` |
| Join | `string.Join(", ", parts)` |
| Replace | `s.Replace("a", "b")` |
| Substring | `s.Substring(0, 5)` or `s[..5]` |
| Trim | `s.Trim()` / `TrimStart()` / `TrimEnd()` |
| Contains | `s.Contains("foo")` |
| StartsWith / EndsWith | `s.StartsWith("a")` |
| ToUpper / ToLower | use `Invariant` variants for non-UI |
| StringBuilder | `var sb = new StringBuilder();` |

---

## Core Concept

`string` is a **reference type but behaves like a value** — equality compares characters, and every "modification" returns a new string (the original is unchanged). This makes strings safe to share but expensive to mutate.

For more than ~5–10 concatenations in a hot path, use `StringBuilder` — it grows an internal buffer and avoids producing intermediate string objects.

For very high-performance scenarios (parsing logs, network protocols), use `ReadOnlySpan<char>` to slice strings without allocating — see [[08 - Span and Memory Types]].

---

## Syntax & API

### Literals
```csharp
string a = "Hello\nWorld";              // \n is newline
string b = @"C:\Users\me";              // verbatim — no escapes
string c = $"Total: {price:C2}";        // interpolated, format spec
string d = $@"Path: {dir}\file.txt";    // verbatim + interpolation

// Raw string literal (C# 11+) — no escaping at all
string json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;

// Raw + interpolation: more $ = more {} needed to escape
string raw = $$"""{ "value": {{x}} }""";
```

### Building strings
```csharp
// BAD for many ops
string result = "";
for (int i = 0; i < 1000; i++) result += i;     // O(n²) allocations

// GOOD
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++) sb.Append(i);
string result = sb.ToString();

// Also good for known-size joins
string csv = string.Join(",", values);
```

### Comparison
```csharp
// Ordinal — byte-by-byte, fast, deterministic — DEFAULT for code
bool eq = string.Equals(a, b, StringComparison.Ordinal);
bool eqIgnore = string.Equals(a, b, StringComparison.OrdinalIgnoreCase);

// Culture — for UI sorting in user's language
int order = string.Compare(a, b, StringComparison.CurrentCulture);
```

### Splitting and joining
```csharp
string csv = "alice,bob,carol";
string[] names = csv.Split(',');
string[] noEmpty = csv.Split(',', StringSplitOptions.RemoveEmptyEntries);

string back = string.Join(", ", names);
```

### Formatting
```csharp
double price = 1234.5;
DateTime now = DateTime.Now;

Console.WriteLine($"{price:C}");            // $1,234.50
Console.WriteLine($"{price:F2}");           // 1234.50
Console.WriteLine($"{price:N0}");           // 1,235
Console.WriteLine($"{price:P1}");           // percent
Console.WriteLine($"{now:yyyy-MM-dd}");     // 2025-01-15
Console.WriteLine($"{42,5}");               // "   42" (right-aligned, width 5)
Console.WriteLine($"{42,-5}|");             // "42   |" (left-aligned)
```

### Slicing & index from end
```csharp
string s = "Hello, World!";
char first = s[0];                  // 'H'
char last  = s[^1];                 // '!'  (^ = from end)
string hello = s[..5];              // "Hello"
string world = s[7..12];            // "World"
```

---

## Common Patterns

```csharp
// Pattern: case-insensitive comparison
if (input.Equals("YES", StringComparison.OrdinalIgnoreCase)) { /* ... */ }
```

```csharp
// Pattern: null-or-empty / null-or-whitespace check
if (string.IsNullOrWhiteSpace(input)) return;
```

```csharp
// Pattern: build a structured log line with StringBuilder
var sb = new StringBuilder(256);
sb.Append('[').Append(DateTime.UtcNow.ToString("o")).Append("] ");
sb.Append(level).Append(' ').Append(message);
return sb.ToString();
```

---

## Gotchas & Tips

- **`==` works for strings** because `string` overrides `Equals` — but it's culture-dependent in some corners. Use explicit `StringComparison.Ordinal` for safety.
- **`ToLower()` and `ToUpper()` use the current culture** — Turkish "I" famously breaks this. Use `ToLowerInvariant()` / `ToUpperInvariant()` for code, never for UI.
- **String interpolation calls `.ToString()`** on each placeholder, allocating. In hot logging paths, use structured logging frameworks (see [[18 - Logging]]).
- **`string.Empty` ≡ `""`** — they refer to the same interned instance.
- **`string` is interned** for literals — `"abc" == "abc"` may be reference-equal, but don't rely on this.
- **`Substring` allocates** — `AsSpan(start, length)` doesn't. Use spans for parsing.

---

## See Also

- [[02 - CSharp Basics]]
- [[08 - Span and Memory Types]]
- [[12 - Serialization]]
