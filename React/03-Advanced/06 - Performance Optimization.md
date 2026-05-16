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

### Lighthouse metrics at a glance

| Metric | What it measures | Good | Needs work | Poor |
| ------ | ---------------- | ---- | ---------- | ---- |
| **LCP** (Largest Contentful Paint) | When the biggest above-the-fold element paints | ≤ 2.5s | 2.5–4.0s | > 4.0s |
| **INP** (Interaction to Next Paint) | Worst event-handler latency in the session (replaced FID in 2024) | ≤ 200ms | 200–500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | How much visible content shifts unexpectedly | ≤ 0.1 | 0.1–0.25 | > 0.25 |
| **FCP** (First Contentful Paint) | First text/image painted | ≤ 1.8s | 1.8–3.0s | > 3.0s |
| **TBT** (Total Blocking Time) | Main-thread blocking between FCP and TTI | ≤ 200ms | 200–600ms | > 600ms |
| **Speed Index** | How quickly visible content is populated | ≤ 3.4s | 3.4–5.8s | > 5.8s |
| **TTFB** (Time to First Byte) | Server response time | ≤ 800ms | 0.8–1.8s | > 1.8s |

> LCP, INP, CLS are the **Core Web Vitals** that affect Google ranking. The rest are diagnostic-only.

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

## Optimizing for Lighthouse — metric by metric

Run Lighthouse from the **Chrome DevTools → Lighthouse panel** (or the standalone extension). Use **Mobile + Simulated throttling** for the default audit — desktop scores hide regressions that hurt real users. Run on a **production build** (`vite build && vite preview`) — dev builds are intentionally slow and skew every number.

### LCP — Largest Contentful Paint

The hero image, headline, or biggest visible block. In React SPAs, LCP is usually killed by **late hydration** of the element that holds it.

React-specific fixes:

- **Server-render the LCP element** (RSC, Next.js, Remix). CSR shows a blank screen until JS arrives — that's a 1–3s LCP tax on its own.
- **Preload the LCP image** in the document head — `<link rel="preload" as="image" href="/hero.webp" fetchpriority="high">`.
- Set **`fetchpriority="high"`** on the hero `<img>`; let everything else be `loading="lazy"`.
- **Do not lazy-load the LCP element.** `React.lazy` + Suspense around the hero adds a chunk waterfall.
- **Self-host critical fonts** with `font-display: swap` and preload the `.woff2` — webfonts often gate text-LCP.
- **Inline above-the-fold CSS** at build time (Vite plugins, Next.js does it automatically).

### INP — Interaction to Next Paint

Replaced FID in March 2024. Measures the **worst** event-handler latency over the whole session — one slow click ruins your score.

React-specific fixes:

- Wrap heavy state updates in **`startTransition`** so the click handler returns immediately (see snippet below).
- Use **`useDeferredValue`** for derived expensive renders driven by typing.
- **Move CPU-heavy work to a Web Worker** (parsing, fuzzy search, markdown render).
- **Virtualize long lists** — clicking a row that triggers a 5000-item re-render tanks INP.
- **Avoid `setState` chains inside the same handler** that each trigger their own render. Batch into one update.
- **Audit third-party scripts** — analytics/ads often dominate INP. Defer with `<script async defer>` or load post-idle.

```tsx
const [isPending, startTransition] = useTransition();
const onSearch = (q: string) => {
  setQuery(q); // urgent — input stays responsive
  startTransition(() => setResults(filter(items, q))); // non-urgent
};
```

### CLS — Cumulative Layout Shift

Content jumping after first paint. React's hydration mismatches and lazy-loaded images are the usual culprits.

React-specific fixes:

- **Always set explicit `width` and `height`** on `<img>` and `<iframe>` (or `aspect-ratio` in CSS) so the browser reserves space.
- **Reserve space for async content** with skeleton placeholders of the same dimensions — don't `null` then suddenly render.
- **Avoid inserting banners/ads above existing content** after mount. Mount them in a fixed-height slot.
- **Watch SSR/CSR hydration mismatches** — they cause a flash + reflow. Keep server and client output identical; use `useEffect` for client-only content (theme, locale-aware date).
- **`font-display: optional`** or preloaded fonts prevent FOIT/FOUT layout swaps.

### FCP — First Contentful Paint

When *any* text/image first paints. Bottlenecked by render-blocking JS and CSS.

React-specific fixes:

- **Code-split per route** with `React.lazy` + Suspense (or framework router).
- **Tree-shake** — verify with `vite build --report` or `rollup-plugin-visualizer`. Drop moment.js → date-fns/dayjs, lodash → lodash-es with named imports.
- **Defer non-critical JS** (`type="module"` is async by default, but third-party `<script>` often is not).
- **Server-render the shell** so HTML arrives painted.

### TBT — Total Blocking Time

Sum of main-thread blocks > 50ms during load. This is the lab-side proxy for INP.

React-specific fixes:

- **Smaller bundles** — every 100KB of JS parses for ~100ms on mid-tier mobile.
- **Hydration cost**: React 18 streaming SSR + selective hydration (or RSC, where most components never hydrate) cuts TBT dramatically.
- **Split vendor chunks** so the framework chunk caches across deploys.
- **Avoid top-level work in modules** — heavy `const data = compute()` at import time blocks the main thread before React even mounts.

### Speed Index

Visual progress over time. Improves when LCP, FCP, and CLS improve — there's no separate lever.

### TTFB — Time to First Byte

Server-side, not React. But it caps every other metric.

Fixes:

- Cache HTML at the CDN edge (Vercel/Netlify/Cloudflare).
- For RSC/SSR, use **streaming** so the first byte is sent before the full render finishes.
- Avoid waterfall data fetches on the server — parallelize with `Promise.all`.

### Lighthouse "Opportunities" → React translation

| Lighthouse suggestion | React-side action |
| --------------------- | ----------------- |
| "Reduce unused JavaScript" | Route-level `React.lazy`, tree-shake, drop dead deps |
| "Properly size images" | `srcSet` + `sizes`, modern formats (AVIF/WebP), Next.js `<Image>` |
| "Preconnect to required origins" | `<link rel="preconnect" href="https://api.example.com">` in `<head>` |
| "Avoid enormous network payloads" | Code-split, compress (brotli), strip source maps from prod |
| "Minimize main-thread work" | RSC, transitions, workers, virtualization |
| "Reduce JavaScript execution time" | React Compiler, memoization where profiler shows waste |
| "Avoid long main-thread tasks" | `startTransition`, `scheduler.yield()` if available |
| "Eliminate render-blocking resources" | Inline critical CSS, `async`/`defer` on third-party scripts |
| "Serve images in next-gen formats" | AVIF/WebP via `<picture>` or framework image component |
| "Ensure text remains visible during webfont load" | `font-display: swap` + preload `.woff2` |

### Tying Lighthouse to field data

Lighthouse is **lab** data — one synthetic run on your machine. Always pair it with **field** data (real users):

```tsx
import { onCLS, onINP, onLCP, onFCP, onTTFB } from "web-vitals";

const send = (m: { name: string; value: number; id: string }) =>
  navigator.sendBeacon("/vitals", JSON.stringify(m));

onCLS(send);
onINP(send);
onLCP(send);
onFCP(send);
onTTFB(send);
```

Send to Sentry Performance, Vercel Analytics, or your own backend. A green Lighthouse score with a red field INP means your test environment isn't representative.

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
