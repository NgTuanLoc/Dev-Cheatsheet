---
tags: [react, intermediate, hooks]
aliases: [useRef, forwardRef, Refs]
level: Intermediate
---

# useRef and forwardRef

> **One-liner**: `useRef` gives you a **mutable container that doesn't trigger re-renders** — used for DOM nodes, timers, and "latest value" boxes; `forwardRef` lets a parent attach a ref through a custom component.

---

## Quick Reference

| Item | Syntax |
|------|--------|
| Create ref | `const ref = useRef<HTMLInputElement>(null)` |
| Attach to DOM | `<input ref={ref} />` |
| Read DOM | `ref.current?.focus()` |
| Mutable value (no render) | `const id = useRef(0); id.current++` |
| Forward ref through component | `forwardRef<Element, Props>((props, ref) => ...)` (React ≤18) |
| React 19+ | `ref` is a regular prop — no `forwardRef` needed |
| Expose imperative API | `useImperativeHandle(ref, () => ({ focus: () => ... }))` |
| Callback ref | `<div ref={(node) => { ... }}>` |

---

## Core Concept

`useState` triggers a re-render when it changes. `useRef` does **not**. It returns a stable `{ current: value }` object that survives re-renders. Mutating `ref.current` is just JS — React doesn't know or care.

Two common uses:

1. **DOM access** — attach to an element via `ref={myRef}`. After commit, `myRef.current` points to the DOM node. Use for focus, measuring, scroll, third-party libs that need a node.
2. **Mutable instance variable** — store a timer ID, latest value, previous value, or any "instance" data without causing renders.

`forwardRef` (React ≤18) is needed because **component props don't include `ref`**. To let a parent attach a ref to your custom component's underlying element, you wrap with `forwardRef`. **In React 19, `ref` is a regular prop** and `forwardRef` is no longer needed.

---

## Syntax & API

### DOM ref — focus on mount

```tsx
import { useRef, useEffect } from "react";

function AutoFocusInput() {
  const ref = useRef<HTMLInputElement>(null);

  useEffect(() => {
    ref.current?.focus();
  }, []);

  return <input ref={ref} />;
}
```

### Mutable value across renders (no re-render)

```tsx
function Stopwatch() {
  const idRef = useRef<number | null>(null);
  const [elapsed, setElapsed] = useState(0);

  const start = () => {
    if (idRef.current !== null) return;
    const startedAt = Date.now() - elapsed;
    idRef.current = window.setInterval(() => {
      setElapsed(Date.now() - startedAt);
    }, 50);
  };

  const stop = () => {
    if (idRef.current !== null) clearInterval(idRef.current);
    idRef.current = null;
  };

  return (
    <>
      <p>{elapsed}ms</p>
      <button onClick={start}>start</button>
      <button onClick={stop}>stop</button>
    </>
  );
}
```

### Storing the previous value

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```

### `forwardRef` (React ≤18)

```tsx
import { forwardRef } from "react";

type InputProps = React.ComponentPropsWithoutRef<"input"> & { label: string };

export const LabeledInput = forwardRef<HTMLInputElement, InputProps>(
  function LabeledInput({ label, ...props }, ref) {
    return (
      <label>
        <span>{label}</span>
        <input ref={ref} {...props} />
      </label>
    );
  }
);

// Parent
const inputRef = useRef<HTMLInputElement>(null);
<LabeledInput ref={inputRef} label="Name" />;
```

### React 19 — `ref` is just a prop

```tsx
type InputProps = { ref?: React.Ref<HTMLInputElement>; label: string };

function LabeledInput({ ref, label, ...rest }: InputProps) {
  return (
    <label>
      <span>{label}</span>
      <input ref={ref} {...rest} />
    </label>
  );
}
```

### `useImperativeHandle` — expose a custom API to parents

```tsx
type Handle = { focus: () => void; clear: () => void };

const FancyInput = forwardRef<Handle, { defaultValue?: string }>(
  function FancyInput({ defaultValue = "" }, ref) {
    const inputRef = useRef<HTMLInputElement>(null);
    const [value, setValue] = useState(defaultValue);

    useImperativeHandle(ref, () => ({
      focus: () => inputRef.current?.focus(),
      clear: () => setValue(""),
    }), []);

    return <input ref={inputRef} value={value} onChange={e => setValue(e.target.value)} />;
  }
);

// Parent
const ref = useRef<Handle>(null);
<FancyInput ref={ref} />;
<button onClick={() => ref.current?.clear()}>Clear</button>
```

### Callback ref — for setup/teardown logic on attach

```tsx
function MeasuredBox() {
  const [width, setWidth] = useState(0);

  return (
    <div ref={(node) => {
      if (node) setWidth(node.getBoundingClientRect().width);
    }}>
      width: {width}px
    </div>
  );
}
```

---

## Common Patterns

```tsx
// Pattern: expose imperative scroll-to API
const Carousel = forwardRef<{ scrollNext: () => void }, Props>((props, ref) => {
  const trackRef = useRef<HTMLDivElement>(null);
  useImperativeHandle(ref, () => ({
    scrollNext: () => trackRef.current?.scrollBy({ left: 200, behavior: "smooth" }),
  }), []);
  return <div ref={trackRef}>...</div>;
});

// Pattern: combine multiple refs (own ref + forwarded ref)
function useCombinedRefs<T>(...refs: Array<React.Ref<T> | undefined>) {
  return useCallback((node: T | null) => {
    refs.forEach(ref => {
      if (typeof ref === "function") ref(node);
      else if (ref != null) (ref as React.MutableRefObject<T | null>).current = node;
    });
  }, refs);
}
```

---

## Gotchas & Tips

- **Don't read/write `ref.current` during render.** It's not reactive — React won't know to update. Use it in effects, handlers, callbacks.
- **Refs survive across renders, not across mounts.** If a component unmounts and remounts, you get a fresh ref.
- **Setting `ref.current` doesn't cause a re-render.** That's the whole point — but it means UI based on the ref needs separate state too.
- **`useRef(null)` for DOM**, `useRef<number | null>(null)` for IDs etc. Initialize to a sensible default.
- **`forwardRef` is going away in React 19** — `ref` becomes a regular prop. Old code keeps working.
- **`useImperativeHandle` is a smell** — most things should be done with props. Only use when an imperative API is genuinely needed (focus, scroll, play/pause).
- **Callback refs run on attach AND detach** (with `null`). Useful for measurement libraries.

---

## See Also

- [[10 - useEffect Basics]]
- [[17 - Refs and DOM]]
- [[19 - TypeScript with React]]
