---
tags: [dotnet, beginner, csharp]
aliases: [List, Dictionary, HashSet, Generic Collections]
level: Beginner
---

# Collections

> **One-liner**: Generic collections in `System.Collections.Generic` cover 99% of needs — pick `List<T>` for ordered, `Dictionary<K,V>` for keyed lookup, `HashSet<T>` for uniqueness, `Queue<T>`/`Stack<T>` for FIFO/LIFO.

---

## Quick Reference

| Collection | Lookup | Insert | Order | Use case |
|------------|--------|--------|-------|----------|
| `T[]` (array) | O(1) by index | fixed size | preserved | known-size data |
| `List<T>` | O(1) by index | O(1) amortized append | preserved | dynamic ordered list |
| `LinkedList<T>` | O(n) | O(1) at known node | preserved | many middle inserts |
| `Dictionary<K,V>` | O(1) by key | O(1) | unordered | keyed lookup |
| `SortedDictionary<K,V>` | O(log n) | O(log n) | by key | ordered keys |
| `HashSet<T>` | O(1) contains | O(1) | unordered | uniqueness |
| `SortedSet<T>` | O(log n) | O(log n) | sorted | unique + sorted |
| `Queue<T>` | n/a | O(1) enqueue/dequeue | FIFO | task queues |
| `Stack<T>` | n/a | O(1) push/pop | LIFO | undo, recursion |
| `IReadOnlyList<T>` | O(1) | n/a | preserved | exposing read-only API |

---

## Core Concept

**Always prefer generic collections** (`List<T>`, `Dictionary<K,V>`) over the legacy non-generic ones (`ArrayList`, `Hashtable`) — they're type-safe, faster, and avoid boxing.

`List<T>` is backed by an array that doubles when full — append is amortized O(1). `Dictionary<K,V>` is a hash table; key lookups are constant time but require a good `GetHashCode`/`Equals` (records and value types get this right by default).

For collections you don't intend to mutate after building, use `IReadOnlyList<T>` / `IReadOnlyDictionary<K,V>` in public APIs to express intent.

---

## Syntax & API

### Array
```csharp
int[] nums = { 1, 2, 3, 4, 5 };
int[] zeros = new int[10];        // length 10, all zero

nums[0] = 99;
Console.WriteLine(nums.Length);
```

### List<T>
```csharp
var list = new List<int> { 1, 2, 3 };
list.Add(4);
list.AddRange(new[] { 5, 6 });
list.Insert(0, 0);
list.Remove(3);                   // remove first occurrence of value
list.RemoveAt(0);                 // remove by index
bool has = list.Contains(2);
int idx = list.IndexOf(4);
list.Sort();
list.Reverse();
list.Clear();
```

### Dictionary<K,V>
```csharp
var ages = new Dictionary<string, int>
{
    ["Alice"] = 30,
    ["Bob"]   = 25
};

ages["Carol"] = 28;                    // add or update
bool exists = ages.ContainsKey("Alice");
if (ages.TryGetValue("Dave", out int age))
    Console.WriteLine(age);

foreach (var (name, value) in ages)
    Console.WriteLine($"{name}: {value}");

ages.Remove("Bob");
```

### HashSet<T>
```csharp
var seen = new HashSet<string>();
bool added = seen.Add("apple");        // true
added = seen.Add("apple");             // false (duplicate)

var a = new HashSet<int> { 1, 2, 3 };
var b = new HashSet<int> { 2, 3, 4 };

a.UnionWith(b);          // a = {1,2,3,4}
a.IntersectWith(b);      // a = {2,3,4}
a.ExceptWith(b);         // a = {} (after intersect)
```

### Queue<T> & Stack<T>
```csharp
var queue = new Queue<string>();
queue.Enqueue("first");
queue.Enqueue("second");
string next = queue.Dequeue();         // "first"
string peek = queue.Peek();            // doesn't remove

var stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
int top = stack.Pop();                 // 2
```

### Collection expressions (C# 12+)
```csharp
List<int>   list  = [1, 2, 3];
int[]       arr   = [4, 5, 6];
HashSet<int> set  = [1, 2, 2, 3];
List<int>   merged = [..list, ..arr, 99];   // spread
```

---

## Common Patterns

```csharp
// Pattern: count occurrences with Dictionary
var counts = new Dictionary<string, int>();
foreach (var word in words)
{
    counts[word] = counts.GetValueOrDefault(word) + 1;
}
```

```csharp
// Pattern: deduplicate with HashSet
var unique = new HashSet<int>(numbers);
```

```csharp
// Pattern: read-only public, mutable internal
public class Order
{
    private readonly List<OrderItem> _items = new();
    public IReadOnlyList<OrderItem> Items => _items;
    public void AddItem(OrderItem item) => _items.Add(item);
}
```

---

## Gotchas & Tips

- **Mutating a collection during `foreach` throws** `InvalidOperationException`. Iterate a copy (`.ToList()`) or use a `for` with index.
- **`Dictionary[key]` throws `KeyNotFoundException`** if the key is missing — prefer `TryGetValue` or `GetValueOrDefault`.
- **`List<T>.Capacity` ≠ `Count`** — `Count` is current items, `Capacity` is allocated slots. Use `EnsureCapacity` if you know the final size to avoid reallocations.
- **Don't use `LinkedList<T>`** unless you have measured a real need — cache locality of `List<T>` usually wins.
- **For thread-safe collections** see [[08 - Synchronization Primitives]] (`ConcurrentDictionary`, `ConcurrentBag`, etc.).
- **`Array.Length` is `int`** — for huge arrays use `LongLength` or `Span<T>`.

---

## See Also

- [[10 - LINQ Basics]]
- [[02 - Generics]]
- [[08 - Synchronization Primitives]]
