---
tags: [react, intermediate, hooks, performance]
aliases: [useMemo, useCallback, Memoization]
level: Intermediate
---

# useMemo and useCallback

> **One-liner**: `useMemo` caches a computed **value**; `useCallback` caches a **function reference** — both prevent unnecessary work or re-renders, but **don't reach for them by default** — measure first.

---

## Quick Reference

| Hook | Returns | When to use |
|------|---------|-------------|
| `useMemo(fn, deps)` | Cached return value of `fn()` | Expensive calculations; stable object refs for deps |
| `useCallback(fn, deps)` | Cached function reference | Passing handlers to memoized children |
| `React.memo(Component)` | Memoized component (skips re-render if props equal) | Pure presentational components in hot paths |

| Don't memoize | Reason |
|---------------|--------|
| Cheap calculations | Memoization itself has cost — likely net loss |
| Functions never passed down | Useless without a memoized consumer |
| Objects passed to non-memoized children | Same — child re-renders anyway |

---

## Core Concept

React re-runs your component on every render. By default, that's fine — function calls and object creation are cheap. **Memoization** caches a value or function across renders so it isn't recomputed/recreated when deps haven't changed.

There are two reasons to memoize:

1. **Skip expensive work.** A 50ms compute on every keystroke is bad; cache it with `useMemo` keyed on the inputs.
2. **Preserve referential equality.** If you pass a new object/function to a memoized child every render, the child re-renders anyway because its props are "different" (new reference). `useMemo`/`useCallback` give you a stable reference.

The cost of memoization is non-zero (deps comparison, cache storage). Adding `useMemo` everywhere often makes apps **slower**. Profile first ([[06 - Performance Optimization]]). React 19's compiler will do this automatically — but until then, memoize only where it pays off.

---

## Syntax & API

### `useMemo` — cache a computed value

```tsx
import { useMemo } from "react";

function ProductList({ products, query }: Props) {
  const filtered = useMemo(
    () => products.filter(p => p.name.includes(query)),
    [products, query],   // recompute only when these change
  );

  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

### `useCallback` — cache a function reference

```tsx
const handleSelect = useCallback((id: string) => {
  setSelected(id);
}, []); // stable forever (no deps)

return <List items={items} onSelect={handleSelect} />;
```

### `React.memo` — skip re-render when props are shallow-equal

```tsx
import { memo } from "react";

type RowProps = { item: Item; onSelect: (id: string) => void };

export const Row = memo(function Row({ item, onSelect }: RowProps) {
  console.log("render row", item.id);
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});
```

> `React.memo` only helps if the parent passes **stable** props. Without `useCallback` for `onSelect`, every parent render gives `Row` a new function → re-renders anyway.

### Putting it together

```tsx
function App() {
  const [items, setItems] = useState<Item[]>(initial);
  const [filter, setFilter] = useState("");

  // Stable function across renders
  const handleSelect = useCallback((id: string) => {
    setItems(prev => prev.map(i => i.id === id ? { ...i, selected: true } : i));
  }, []);

  // Cached filtered result
  const visible = useMemo(
    () => items.filter(i => i.name.includes(filter)),
    [items, filter],
  );

  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <ul>
        {visible.map(item => (
          <Row key={item.id} item={item} onSelect={handleSelect} />
        ))}
      </ul>
    </>
  );
}
```

---

## Common Patterns

```tsx
// Pattern: stable context value (avoid re-rendering all consumers)
const value = useMemo(() => ({ user, setUser }), [user]);
return <UserCtx.Provider value={value}>{children}</UserCtx.Provider>;

// Pattern: memoize derived selector
const isAdmin = useMemo(() => user.roles.includes("admin"), [user.roles]);

// Pattern: useCallback for an event handler attached in an effect
const onResize = useCallback(() => setSize(window.innerWidth), []);
useEffect(() => {
  window.addEventListener("resize", onResize);
  return () => window.removeEventListener("resize", onResize);
}, [onResize]);
```

---

## Gotchas & Tips

- **Memoization isn't free.** Deps comparison, cache, and the closure all cost. For trivial work, skip it.
- **`React.memo` does shallow prop comparison.** New objects/arrays/functions = different. That's why you also need `useMemo`/`useCallback` upstream.
- **Custom comparator: `memo(Component, (prev, next) => ...)`** — return `true` to skip render. Easy to get wrong; rarely needed.
- **`useMemo` is not a guarantee** — React may discard the cache (e.g., on memory pressure). Don't rely on it for correctness, only perf.
- **`useCallback` with empty deps is the most-stable form** — but only safe if the function reads state via setters' functional form.
- **React Compiler (RC in React 19) auto-memoizes** — when stable, manual `useMemo`/`useCallback` calls become unnecessary.
- **Don't memoize JSX elements via `useMemo`** unless the child is *very* expensive — the parent still re-renders to compute them. `React.memo` on the child is usually the right tool.
- **Profile, don't guess.** Open React DevTools → Profiler tab. Measure before and after.

---

## See Also

- [[06 - Performance Optimization]]
- [[05 - useContext]]
- [[01 - React Internals]]
