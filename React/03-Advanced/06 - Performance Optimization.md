---
tags: [react, advanced, performance]
aliases: [Performance, React Performance]
level: Advanced
---

# Performance Optimization

> **One-liner**: Optimize what's actually slow — measure with the **React Profiler** and browser **performance** tab first, then attack with memoization, virtualization, code splitting, and (in 2025) the React Compiler.

---

## Quick Reference

| Bottleneck | Fix |
|------------|-----|
| Big bundle / slow first paint | Code splitting, tree-shaking, RSC |
| Long lists | List virtualization (`react-window`, `@tanstack/react-virtual`) |
| Slow expensive calc on every render | `useMemo` |
| Memoized child re-renders | `useCallback` + stable props |
| Whole tree re-renders on every keystroke | `useDeferredValue`, `startTransition` |
| Context broadcasts cause re-renders everywhere | Split contexts, or use Zustand selectors |
| Image LCP | `<img loading="lazy">`, responsive `srcset`, modern formats |
| 3rd-party JS hurts INP | Defer/lazy-load, use Web Workers |
| TBT spikes from many tiny updates | Batch via transition, throttle/debounce |

| Profiling tool | Use |
|----------------|------|
| React DevTools Profiler | Component-level: who rendered, why, how long |
| Chrome Performance tab | Frame-by-frame; main-thread time, INP, LCP |
| Lighthouse | Overall report, web-vitals, suggestions |
| `web-vitals` library | Field RUM data |
| `why-did-you-render` | Logs unnecessary re-renders during dev |

---

## Core Concept

Performance work in React is unmistakably **measure → fix → measure**. Intuition is wrong half the time. The Profiler shows what actually rendered and how long each render took; until you have that data, you're guessing.

The hierarchy of payoff (biggest first, in most apps):
1. **Bundle size** — fewer JS bytes downloaded/parsed/executed.
2. **Network round-trips** — RSC removes them entirely; TanStack Query dedupes.
3. **List virtualization** — never render 10k DOM nodes.
4. **Memoization** — only after a profile shows wasted re-renders.
5. **Concurrent features** — keep input responsive while heavy work runs.
6. **Micro-perf** (algorithmic improvements) — almost always the smallest win.

The 2025 game-changer is the **React Compiler** (RC for React 19). It auto-memoizes components and hooks at build time, eliminating most manual `useMemo`/`useCallback`. Adopt it gradually per file/route once stable.

---

## Syntax & API

### React DevTools Profiler — record a session

```text
1. Open React DevTools → Profiler tab
2. Click "Record"
3. Interact with the app
4. Click "Stop"
5. Look at the flamegraph:
   - Wide bars = expensive components
   - "Why did this render?" panel shows props/state/hook changes
   - "Ranked" view sorts by duration
```

### Web Vitals in production (RUM)

```ts
import { onCLS, onINP, onLCP } from "web-vitals";

onCLS(metric => beacon("cls", metric));
onINP(metric => beacon("inp", metric));
onLCP(metric => beacon("lcp", metric));
```

### List virtualization — `@tanstack/react-virtual`

```tsx
import { useVirtualizer } from "@tanstack/react-virtual";

function BigList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null);

  const v = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 32,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      <div style={{ height: v.getTotalSize(), position: "relative" }}>
        {v.getVirtualItems().map(virtual => (
          <div
            key={virtual.key}
            style={{
              position: "absolute",
              top: 0,
              left: 0,
              right: 0,
              height: virtual.size,
              transform: `translateY(${virtual.start}px)`,
            }}
          >
            {items[virtual.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Smart memoization

```tsx
const Row = memo(function Row({ item, onSelect }: { item: Item; onSelect: (id: string) => void }) {
  return <li onClick={() => onSelect(item.id)}>{item.name}</li>;
});

function List({ items }: { items: Item[] }) {
  const onSelect = useCallback((id: string) => doThing(id), []); // stable
  return <ul>{items.map(i => <Row key={i.id} item={i} onSelect={onSelect} />)}</ul>;
}
```

### Defer non-urgent updates

```tsx
const [text, setText] = useState("");
const deferred = useDeferredValue(text);

return (
  <>
    <input value={text} onChange={e => setText(e.target.value)} />
    <ExpensiveResults query={deferred} />
  </>
);
```

### Web Workers for CPU-heavy work

```ts
// worker.ts
self.onmessage = (e) => {
  const result = expensive(e.data);
  self.postMessage(result);
};

// component.tsx
useEffect(() => {
  const worker = new Worker(new URL("./worker.ts", import.meta.url), { type: "module" });
  worker.postMessage(input);
  worker.onmessage = e => setResult(e.data);
  return () => worker.terminate();
}, [input]);
```

---

## Common Patterns

```tsx
// Pattern: stable Context value
const value = useMemo(() => ({ user, setUser }), [user]);
return <UserCtx.Provider value={value}>{children}</UserCtx.Provider>;

// Pattern: avoid prop drilling causing re-renders — keep state local
//          or use Zustand selectors that bail out on unchanged slices
const count = useStore(s => s.count);   // only re-renders when count changes

// Pattern: image hints
<img src="/hero.webp" srcSet="/hero@2x.webp 2x" loading="lazy" decoding="async" />
```

---

## Gotchas & Tips

- **Always profile before optimizing.** Real bottlenecks are rarely where you think.
- **Memoization isn't free.** Adding `useMemo`/`useCallback` to every component costs more than it saves.
- **The biggest perf win is usually deleting code** — unused libs, dead routes, oversized images.
- **Virtualize lists over ~50 items**, especially with complex rows.
- **Don't render 10 charts on one screen.** Stagger with intersection-observer-driven mounts.
- **Layout thrashing**: don't read DOM measurements in a loop while writing styles. Batch reads, then writes.
- **`React.memo` + non-stable props = no help.** Profile to confirm the bailout actually triggers.
- **For SSR/RSC apps**, server work is usually the bottleneck, not React render. Profile the server too.
- **INP (Interaction to Next Paint) is the new RUM king.** Slow event handlers and long tasks hurt INP — split work, use transitions.
- **React Compiler will obsolete much of this** for new code. Until adopted, hand-tune sparingly with profiler data.

---

## See Also

- [[01 - React Internals]]
- [[02 - Concurrent Features]]
- [[03 - useMemo and useCallback]]
- [[14 - Code Splitting]]
- [[17 - Monitoring and Errors]]
