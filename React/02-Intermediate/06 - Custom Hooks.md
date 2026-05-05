---
tags: [react, intermediate, hooks]
aliases: [Custom Hooks, useX]
level: Intermediate
---

# Custom Hooks

> **One-liner**: A custom hook is a JavaScript function whose name starts with `use` and which calls other hooks — it bundles stateful logic so multiple components can share it without sharing state.

---

## Quick Reference

| Rule | Why |
|------|-----|
| Name starts with `use` | The lint rule needs it; React DevTools uses it |
| Call hooks only at top level | Same hook order every render |
| Call hooks only from React functions or other hooks | Hook dispatcher must be active |
| Returns whatever — value, tuple, object | Convention: `[value, setter]` for setState-like, object for many things |
| **Each call has its own state** | Two components calling `useToggle()` get independent state |

---

## Core Concept

You write a custom hook to **reuse stateful logic**, not state itself. If three components all need to track window size, instead of duplicating `useState` + `useEffect` three times, extract `useWindowSize()`. Each call still has its own state — the hook just packages the *pattern*.

A custom hook is a regular function that calls other hooks. The "use" prefix isn't magic — it's how the linter (and React DevTools) recognizes it. The **rules of hooks** apply: call them at the top level (not in conditionals or loops), and only from React render functions or other hooks.

Custom hooks are how the React community shares logic. There's no React-specific machinery; just convention and the lint rule.

---

## Syntax & API

### A toggle hook

```tsx
import { useState, useCallback } from "react";

function useToggle(initial = false): [boolean, () => void, (v: boolean) => void] {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(o => !o), []);
  return [on, toggle, setOn];
}

// Usage
function Modal() {
  const [open, toggle] = useToggle(false);
  return (
    <>
      <button onClick={toggle}>{open ? "Close" : "Open"}</button>
      {open && <div>...</div>}
    </>
  );
}
```

### `useLocalStorage` — sync state with localStorage

```tsx
function useLocalStorage<T>(key: string, initial: T) {
  const [value, setValue] = useState<T>(() => {
    const raw = localStorage.getItem(key);
    return raw ? (JSON.parse(raw) as T) : initial;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue] as const;
}

// Usage
const [name, setName] = useLocalStorage("name", "");
```

### `useWindowSize` — subscribe to a browser API

```tsx
function useWindowSize() {
  const [size, setSize] = useState(() => ({
    w: window.innerWidth,
    h: window.innerHeight,
  }));

  useEffect(() => {
    const onResize = () => setSize({ w: window.innerWidth, h: window.innerHeight });
    window.addEventListener("resize", onResize);
    return () => window.removeEventListener("resize", onResize);
  }, []);

  return size;
}
```

### `useDebounced` — delayed value

```tsx
function useDebounced<T>(value: T, ms = 300): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), ms);
    return () => clearTimeout(id);
  }, [value, ms]);

  return debounced;
}

// Usage
const [query, setQuery] = useState("");
const debounced = useDebounced(query, 250);
useEffect(() => { search(debounced); }, [debounced]);
```

### `usePrevious` — remember last render's value

```tsx
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T | undefined>(undefined);
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```

### `useFetch` — light data fetcher (for serious use, prefer TanStack Query)

```tsx
type FetchState<T> =
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error";   error: Error };

function useFetch<T>(url: string): FetchState<T> {
  const [state, setState] = useState<FetchState<T>>({ status: "loading" });

  useEffect(() => {
    const ctrl = new AbortController();
    setState({ status: "loading" });

    fetch(url, { signal: ctrl.signal })
      .then(r => r.json() as Promise<T>)
      .then(data => setState({ status: "success", data }))
      .catch(error => {
        if (error.name !== "AbortError") setState({ status: "error", error });
      });

    return () => ctrl.abort();
  }, [url]);

  return state;
}
```

---

## Common Patterns

```tsx
// Pattern: returning tuple vs object
const [value, setValue] = useToggle();        // tuple — feels like useState
const { isLoading, data, error } = useUser(); // object — many fields

// Pattern: composing hooks
function useUser(id: string) {
  const { data, ...rest } = useQuery(["user", id], () => fetchUser(id));
  const isAdmin = useMemo(() => data?.roles.includes("admin") ?? false, [data]);
  return { user: data, isAdmin, ...rest };
}

// Pattern: taking deps as an argument when needed
function useEventListener<K extends keyof WindowEventMap>(
  event: K,
  handler: (e: WindowEventMap[K]) => void,
) {
  useEffect(() => {
    window.addEventListener(event, handler);
    return () => window.removeEventListener(event, handler);
  }, [event, handler]);
}
```

---

## Gotchas & Tips

- **Two components calling the same hook do NOT share state.** Each call is independent. To share state, lift it to context or a global store.
- **Hooks aren't stored across mounts.** A hook's state lives only as long as the component does.
- **Don't call hooks conditionally.** `if (cond) useState(...)` violates the rules — order must be stable.
- **Don't call hooks from event handlers.** They run during render only.
- **Naming matters**: prefix with `use`. The lint rule (`react-hooks/rules-of-hooks`) only enforces hooks rules for `useX` names.
- **Hook return shape is a contract.** Tuple is fine for 2 items (mirrors `useState`); use an object for 3+.
- **Custom hooks are easy to test.** Use `@testing-library/react`'s `renderHook`.
- **Common community libraries**: `usehooks-ts`, `react-use`, `@uidotdev/usehooks` — well-maintained collections worth grepping before writing your own.

---

## See Also

- [[10 - useEffect Basics]]
- [[01 - useEffect Deep Dive]]
- [[02 - useRef and forwardRef]]
- [[18 - Testing]]
