---
tags: [react, advanced, state-management]
aliases: [Redux Toolkit, Zustand, Jotai, Signals]
level: Advanced
---

# State Management Advanced

> **One-liner**: Past local + Context, the pragmatic 2025 choices are **Zustand** (small global stores), **Jotai** (atoms), **Redux Toolkit** (mature ecosystem, time-travel devtools), and **TanStack Query** (server state) — pick by team familiarity and the shape of your state, not by hype.

---

## Quick Reference

| Library | Model | Boilerplate | Devtools | Sweet spot |
|---------|-------|-------------|----------|------------|
| **Zustand** | Single store, hook selectors | minimal | yes | Small/medium apps; light global state |
| **Jotai** | Atoms (Recoil-like, refined) | minimal | yes | Granular fine-grained state, derived atoms |
| **Redux Toolkit + RTK Query** | Slice + reducer + middleware | medium (vs old Redux: drastically less) | excellent | Large teams, audit trail, time-travel |
| **MobX** | Observables, reactive | low at use, more at setup | yes | OO-heavy codebases; rare in new React |
| **Signals** (Preact / SolidJS) | Fine-grained reactivity | very low | growing | Not yet first-party in React; experimental in libraries |
| **TanStack Query** | Server cache | none | yes | All server-derived data |

---

## Core Concept

Local + Context + Query covers most apps. Past that, "global client state" becomes a category — and the libraries above each take a different position on **how to structure it** and **how to subscribe to slices without re-rendering everything**.

The shared insight: **subscriptions must be selective**. If every consumer re-renders on every store change, you've reinvented Context's worst flaw on top of more code. Modern stores all give you **selectors** — pass a function that picks the slice you care about, and you only re-render when *that* slice's reference changes.

Pick by:
- **Team's existing knowledge** (Redux is universal; Zustand is smaller; Jotai's atom mental model is different).
- **Devtools needs** (RTK has time-travel debugging — invaluable for complex flows).
- **Bundle size** (Zustand and Jotai are tiny; RTK is bigger).
- **Whether you need middleware** (logging, persistence, sagas) — RTK's ecosystem is widest.

---

## Syntax & API

### Zustand — minimal global store

```bash
npm install zustand
```

```tsx
import { create } from "zustand";
import { persist } from "zustand/middleware";

type Store = {
  filter: "all" | "active" | "done";
  setFilter: (f: Store["filter"]) => void;
};

export const useTodoStore = create<Store>()(
  persist(
    set => ({
      filter: "all",
      setFilter: filter => set({ filter }),
    }),
    { name: "todo-filter" },     // localStorage key
  ),
);

// Selectors avoid extra re-renders
function FilterTabs() {
  const filter    = useTodoStore(s => s.filter);
  const setFilter = useTodoStore(s => s.setFilter);
  return (
    <>
      <button onClick={() => setFilter("all")}    aria-pressed={filter === "all"}>All</button>
      <button onClick={() => setFilter("active")} aria-pressed={filter === "active"}>Active</button>
      <button onClick={() => setFilter("done")}   aria-pressed={filter === "done"}>Done</button>
    </>
  );
}
```

### Jotai — atomic state

```bash
npm install jotai
```

```tsx
import { atom, useAtom, useAtomValue } from "jotai";

const countAtom = atom(0);
const doubledAtom = atom(get => get(countAtom) * 2);   // derived

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const doubled = useAtomValue(doubledAtom);
  return (
    <>
      <p>{count} → {doubled}</p>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
    </>
  );
}

// Async derived atom
const userAtom = atom(async () => fetch("/api/user").then(r => r.json()));
const user = useAtomValue(userAtom);    // suspends
```

### Redux Toolkit + RTK Query

```bash
npm install @reduxjs/toolkit react-redux
```

```tsx
import { configureStore, createSlice } from "@reduxjs/toolkit";
import { Provider, useDispatch, useSelector } from "react-redux";

const ui = createSlice({
  name: "ui",
  initialState: { sidebarOpen: true },
  reducers: {
    toggleSidebar(s) { s.sidebarOpen = !s.sidebarOpen; }, // Immer — mutation OK
  },
});

const store = configureStore({ reducer: { ui: ui.reducer } });
type RootState = ReturnType<typeof store.getState>;

function Sidebar() {
  const open = useSelector((s: RootState) => s.ui.sidebarOpen);
  const dispatch = useDispatch();
  return (
    <>
      <button onClick={() => dispatch(ui.actions.toggleSidebar())}>Toggle</button>
      {open && <aside>...</aside>}
    </>
  );
}
```

### RTK Query — Redux's data-fetching layer

```ts
import { createApi, fetchBaseQuery } from "@reduxjs/toolkit/query/react";

export const api = createApi({
  reducerPath: "api",
  baseQuery: fetchBaseQuery({ baseUrl: "/api" }),
  endpoints: builder => ({
    getUser: builder.query<User, string>({ query: id => `/users/${id}` }),
    addUser: builder.mutation<User, NewUser>({
      query: body => ({ url: "/users", method: "POST", body }),
      invalidatesTags: ["User"],
    }),
  }),
  tagTypes: ["User"],
});

export const { useGetUserQuery, useAddUserMutation } = api;
```

### Signals (via libraries — not native React yet)

```ts
// @preact/signals-react example
import { signal, computed } from "@preact/signals-react";

const count = signal(0);
const doubled = computed(() => count.value * 2);

function Counter() {
  return (
    <>
      <p>{count} × 2 = {doubled}</p>
      <button onClick={() => count.value++}>+</button>
    </>
  );
}
// Reads track usage; only components that read the signal re-render.
```

---

## Common Patterns

```tsx
// Pattern: combine multiple stores by domain
const useAuth   = create(...);
const useUI     = create(...);
const useCart   = create(...);
// Co-locate by feature; share via custom hooks if cross-cutting

// Pattern: scoped Zustand store via Context (per-instance state, e.g., per chart)
const ChartCtx = createContext<StoreApi<ChartState> | null>(null);
function ChartProvider({ children }) {
  const [store] = useState(() => createChartStore());
  return <ChartCtx.Provider value={store}>{children}</ChartCtx.Provider>;
}
```

---

## Gotchas & Tips

- **Don't put server data in any of these.** TanStack Query / RTK Query is the right home — it dedupes, retries, and refetches.
- **Selectors must return stable references** — derived objects should be memoized or computed inside the store, not in the selector.
- **`useShallow` (Zustand)** for selectors that return objects/arrays — avoids re-renders when contents are equal.
- **Persistence** is straightforward (Zustand `persist` middleware, Redux `redux-persist`) but careful about serializing class instances or Dates.
- **Redux Toolkit replaces all classic Redux boilerplate.** If you've avoided Redux because of action constants and switch statements — try RTK; it's a different beast.
- **RTK Query overlaps with TanStack Query.** Pick one for server state. RTK Query if you're already in Redux; TanStack otherwise.
- **Signals in React are not first-party** as of late 2025. Watch the React Compiler space — it may obsolete some signal use cases.
- **Don't reach for global state to avoid prop drilling at 2 levels.** Lift first, Context for subtrees, store only when truly cross-cutting.

---

## See Also

- [[12 - State Management]]
- [[05 - useContext]]
- [[11 - TanStack Query]]
