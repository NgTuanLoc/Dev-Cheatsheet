---
tags: [react, index, learning-path]
aliases: [React Roadmap, React Curriculum]
---

# Learning Path

> **Recommended order** to study the vault. Stages assume you know HTML/CSS and basic JavaScript and build to production-grade depth.

---

## Stage 1 — React Fundamentals (1 week)

**Goal**: Build a small interactive UI from scratch — components, state, events, lists.

1. [[01 - React Overview]] — orient the ecosystem
2. [[02 - JSX Basics]]
3. [[03 - Components and Props]]
4. [[04 - State and useState]]
5. [[05 - Event Handling]]
6. [[06 - Conditional Rendering]]
7. [[07 - Lists and Keys]]
8. [[08 - Forms and Inputs]]
9. [[09 - Styling]]
10. [[10 - useEffect Basics]]

**Checkpoint**: Build a TODO app with add/edit/delete, filter by status, persist to `localStorage`.

---

## Stage 2 — Hooks Mastery (1 week)

**Goal**: Reach for the right hook without thinking. The biggest source of React bugs is misuse of `useEffect` and stale closures.

1. [[01 - useEffect Deep Dive]] ← **read first**
2. [[02 - useRef and forwardRef]]
3. [[03 - useMemo and useCallback]]
4. [[04 - useReducer]]
5. [[05 - useContext]]
6. [[06 - Custom Hooks]]

**Checkpoint**: Refactor your TODO app to use `useReducer` for state and a custom `useLocalStorage` hook for persistence.

---

## Stage 3 — Component Patterns (3–5 days)

**Goal**: Stop copy-pasting; compose instead.

1. [[07 - Component Composition]]
2. [[08 - Higher-Order Components]]
3. [[16 - Portals]]
4. [[17 - Refs and DOM]]

---

## Stage 4 — Forms, Data, State (1–2 weeks)

**Goal**: Ship a real CRUD app with API calls, validated forms, and global state.

1. [[10 - Fetching Data]]
2. [[11 - TanStack Query]]
3. [[09 - Forms Advanced]]
4. [[12 - State Management]]
5. [[15 - Error Boundaries]]

**Checkpoint**: A small CRUD app (e.g. notes manager) backed by a REST API, with React Hook Form + Zod, TanStack Query for fetching, and an error boundary on the root.

---

## Stage 5 — Routing & Splitting (3–5 days)

1. [[13 - React Router]]
2. [[14 - Code Splitting]]

**Checkpoint**: Add nested routes, route-level loaders, and lazy-loaded routes to your CRUD app.

---

## Stage 6 — Quality (1 week)

**Goal**: Make your code reviewable.

1. [[19 - TypeScript with React]]
2. [[18 - Testing]]
3. [[20 - Accessibility]]

**Checkpoint**: Convert a feature to TypeScript, write component + integration tests, run axe-core and fix all violations.

---

## Stage 7 — Runtime & Concurrent React (1 week)

**Goal**: Understand what React actually does so you can debug it.

1. [[01 - React Internals]]
2. [[02 - Concurrent Features]]
3. [[03 - Suspense]]
4. [[06 - Performance Optimization]]

---

## Stage 8 — Server Side (1–2 weeks)

**Goal**: Ship rendering on the server. Today this means Next.js or Remix.

1. [[07 - Rendering Strategies]]
2. [[04 - Server Components]]
3. [[05 - Server Actions]]
4. [[08 - Next.js App Router]]
5. [[09 - Remix and React Router 7]]
6. [[11 - Forms at Scale]]

**Checkpoint**: Port your CRUD app to Next.js App Router with Server Components for reads and Server Actions for writes.

---

## Stage 9 — Design & UX (1 week)

1. [[14 - Design Systems]]
2. [[12 - Animations]]
3. [[13 - Internationalization]]
4. [[10 - State Management Advanced]]

---

## Stage 10 — Architecture & Deployment (3–5 days)

1. [[16 - Build and Bundling]]
2. [[17 - Monitoring and Errors]]
3. [[15 - Micro-Frontends]]
4. [[18 - React Native and Cross-Platform]]

---

## Tips for using this path

- **Don't skip Stage 2** — `useEffect` misunderstandings cause most production bugs in React apps.
- **Build the checkpoint at each stage.** Reading without coding doesn't stick.
- **Don't reach for Redux on day one.** Local state + Context covers 80% of apps; learn the global-state tools later in Stage 4.
- **Test as you learn** — adding tests retroactively is painful; write them alongside features after Stage 6.
- Use [[React Master Index]] to jump around; this path is recommended, not mandatory.
