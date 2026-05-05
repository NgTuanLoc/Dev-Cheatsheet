---
tags: [react, intermediate, hooks]
aliases: [DOM Access, Imperative DOM]
level: Intermediate
---

# Refs and DOM

> **One-liner**: Most of the time React owns the DOM, but sometimes you need to reach into it imperatively — to focus, scroll, measure, or hand a node to a non-React library — and that's what refs are for.

---

## Quick Reference

| Need | Tool |
|------|------|
| Focus an input on mount | `useRef` + `useEffect` + `el.focus()` |
| Scroll a list to bottom | `useRef` + `el.scrollTop = el.scrollHeight` |
| Measure box | `useRef` + `getBoundingClientRect()` (or `ResizeObserver`) |
| Watch resize | `ResizeObserver` |
| Watch viewport intersection | `IntersectionObserver` (lazy load, infinite scroll) |
| Hand DOM to a library (Mapbox, D3) | Pass `ref.current` to the lib's `init` |
| Imperative video/audio control | `videoRef.current.play() / pause()` |
| Reset scroll on route change | `useEffect(..., [location])` |

---

## Core Concept

React's value proposition is "stop touching the DOM." But the DOM has irreducibly imperative APIs: focus, scroll, media playback, canvas drawing, third-party widgets. For those, you need a handle to the actual node — that's a **ref**.

A ref pointing to a DOM node:
- Is attached **after commit** — `ref.current` is `null` during render.
- Is **opt-out from reactivity** — changing it doesn't re-render. Combine with `useState` if UI needs to follow.
- Should be used in **effects, handlers, callbacks** — never read during render to *decide what to render*.

For non-React libraries (Mapbox, D3, Chart.js, Lottie): mount on a `<div ref={ref}>`, pass `ref.current` into the library's init in `useEffect`, and clean up in the cleanup function. The library mutates the div directly — React's diff doesn't go inside it.

---

## Syntax & API

### Focus on mount

```tsx
function AutoFocusInput() {
  const ref = useRef<HTMLInputElement>(null);
  useEffect(() => { ref.current?.focus(); }, []);
  return <input ref={ref} />;
}
```

### Scroll to bottom (chat window)

```tsx
function ChatLog({ messages }: { messages: Message[] }) {
  const scroller = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const el = scroller.current;
    if (el) el.scrollTop = el.scrollHeight;
  }, [messages.length]);

  return (
    <div ref={scroller} className="overflow-auto h-96">
      {messages.map(m => <p key={m.id}>{m.text}</p>)}
    </div>
  );
}
```

### Measure with `ResizeObserver`

```tsx
function useSize<T extends HTMLElement>() {
  const ref = useRef<T>(null);
  const [size, setSize] = useState({ w: 0, h: 0 });

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const ro = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      setSize({ w: width, h: height });
    });
    ro.observe(el);
    return () => ro.disconnect();
  }, []);

  return { ref, size };
}

// Usage
const { ref, size } = useSize<HTMLDivElement>();
return <div ref={ref}>{size.w}px wide</div>;
```

### Intersection observer — lazy image / infinite scroll

```tsx
function useInView<T extends HTMLElement>(opts?: IntersectionObserverInit) {
  const ref = useRef<T>(null);
  const [inView, setInView] = useState(false);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const io = new IntersectionObserver(([entry]) => setInView(entry.isIntersecting), opts);
    io.observe(el);
    return () => io.disconnect();
  }, [opts]);

  return { ref, inView };
}
```

### Integrating a non-React library (sketch)

```tsx
function MapView({ center }: { center: [number, number] }) {
  const containerRef = useRef<HTMLDivElement>(null);
  const mapRef = useRef<mapboxgl.Map | null>(null);

  useEffect(() => {
    if (!containerRef.current) return;
    mapRef.current = new mapboxgl.Map({
      container: containerRef.current,
      center,
      zoom: 10,
    });
    return () => mapRef.current?.remove();
  }, []);

  // React-side prop change → call into the library
  useEffect(() => {
    mapRef.current?.setCenter(center);
  }, [center]);

  return <div ref={containerRef} className="h-96" />;
}
```

---

## Common Patterns

```tsx
// Pattern: callback ref — runs on attach AND detach
<div ref={node => {
  if (node) console.log("attached", node);
  else      console.log("detached");
}}>...</div>

// Pattern: combine multiple refs
function useCombined<T>(...refs: React.Ref<T>[]) {
  return useCallback((node: T | null) => {
    refs.forEach(r => {
      if (typeof r === "function") r(node);
      else if (r) (r as React.MutableRefObject<T | null>).current = node;
    });
  }, refs);
}
```

```tsx
// Pattern: useLayoutEffect for measurement before paint
useLayoutEffect(() => {
  const rect = ref.current!.getBoundingClientRect();
  setHeight(rect.height);     // synchronous → no flicker
}, []);
```

---

## Gotchas & Tips

- **`ref.current` is `null` on first render.** Always guard or read in an effect.
- **Don't mutate the DOM where React renders.** React will overwrite your changes on re-render. Mutate inside ref'd containers React doesn't traverse (third-party widgets) or via React-aware APIs (`focus`, `scroll`).
- **Use `useLayoutEffect` for synchronous DOM measurements that drive layout** (otherwise you get a flash of wrong size).
- **Don't store DOM nodes in `useState`.** It causes extra re-renders and serialization issues. `useRef` is right.
- **Strict Mode mounts twice in dev** — make sure cleanup actually disconnects observers / removes nodes.
- **`ResizeObserver` is not in tests by default.** Mock it in Jest/Vitest setup, or use a library that handles it.
- **`forwardRef` (or React 19's prop-style `ref`)** is needed when a parent wants to attach a ref to your custom component's root element.
- **For accessibility focus management**, prefer `react-aria` / `react-focus-lock` over hand-rolled — getting it right is hard.

---

## See Also

- [[02 - useRef and forwardRef]]
- [[16 - Portals]]
- [[20 - Accessibility]]
