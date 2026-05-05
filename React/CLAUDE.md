# CLAUDE.md — React Topic

Topic-specific guidance for the `React/` section of the vault. For shared rules (template, Mermaid, authoring conventions), see the root [CLAUDE.md](../CLAUDE.md).

## What This Topic Is

A structured React teacher-authored cheatsheet/notebook — organized from beginner to advanced, intended for students learning modern React (18+ / 19+) and the surrounding ecosystem (TypeScript, Vite, React Router, TanStack Query, Next.js).

---

## Version & Tooling Policy

- **React 18+ is the baseline**, with React 19 features (Actions, `useActionState`, `useFormStatus`, `use`) marked explicitly when introduced.
- **Function components + hooks only** — class components appear only as a brief historical mention in [[01 - React Overview]].
- **TypeScript-first** — examples are TS by default. Plain JS is shown only where it adds clarity.
- **Vite** is the default dev/build tool. `create-react-app` is deprecated and is mentioned only as a historical note.
- **Bundler-agnostic where possible** — patterns work in Vite, Next.js, Remix unless explicitly tied to one.

When a topic differs significantly across React 18 vs 19, or across Vite vs Next.js, call it out in a callout sub-section:

````markdown
### Framework notes

- **React 18**: Suspense for code-splitting, manual data-fetching with effects.
- **React 19**: Adds Actions, `useActionState`, `useFormStatus`, `use(promise)`.
- **Next.js (App Router)**: Server Components by default; mark client code with `"use client"`.
````

---

## Folder Structure

```
React/
├── CLAUDE.md                    ← this file
├── 00-Index/
│   ├── React Master Index.md    ← single entry point, links to all React notes
│   └── React Learning Path.md   ← recommended reading order per level
├── 01-Beginner/                 (10 notes)
├── 02-Intermediate/             (20 notes)
└── 03-Advanced/                 (18 notes)
```

---

## Tag Taxonomy (React-specific)

Always include `react` plus the level. Add domain tags as relevant.

| Tag | Meaning |
|-----|---------|
| `react` | applied to every React note |
| `jsx` | JSX syntax / templating |
| `hooks` | useState, useEffect, custom hooks |
| `typescript` | TypeScript-with-React specifics |
| `state-management` | local state, context, Redux, Zustand, Jotai |
| `forms` | controlled inputs, React Hook Form, validation |
| `routing` | React Router, Next.js routing |
| `data-fetching` | fetch patterns, TanStack Query, SWR |
| `performance` | memoization, profiling, virtualization |
| `testing` | Vitest, React Testing Library, Playwright |
| `accessibility` | ARIA, semantic HTML, keyboard nav |
| `ssr` | server-side rendering, hydration |
| `rsc` | React Server Components |
| `nextjs` | Next.js-specific |
| `remix` | Remix / React Router 7 |
| `styling` | CSS modules, Tailwind, CSS-in-JS |
| `animation` | Framer Motion, transitions |
| `architecture` | component design, composition, design systems |
| `build` | Vite, Webpack, esbuild, bundling |
| `devops` | deploy, monitoring, error tracking |

---

## Code-Block Conventions

| Lang tag | Use for |
|----------|---------|
| `tsx` | TypeScript + JSX (the default for component code) |
| `ts` | TypeScript without JSX (hooks, utilities, types) |
| `jsx` | Plain JavaScript + JSX (only when TS adds noise) |
| `js` | Plain JavaScript without JSX |
| `bash` | CLI: `npm`, `pnpm`, `npx vite`, `next dev` |
| `json` | `package.json`, `tsconfig.json`, config files |
| `css` | CSS, CSS modules, Tailwind classes |
| `html` | Static HTML, `index.html` |
| `yaml` | GitHub Actions, Vercel/Netlify config |
| `mermaid` | diagrams |

Examples must be **runnable** — student should be able to drop them into a fresh Vite + React + TS project and see them work. Prefer `tsx` over `jsx` unless the example is intentionally about plain JS.

---

## Topic Coverage Map

### Beginner (01-Beginner) — 10 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | React Overview | What React is, virtual DOM, declarative vs imperative, components, ecosystem map |
| 02 | JSX Basics | JSX syntax, expressions, attributes, children, fragments, JSX vs HTML differences |
| 03 | Components and Props | Function components, props, destructuring, children prop, composition basics |
| 04 | State and useState | useState, immutable updates, functional updaters, lazy init, multiple state vars |
| 05 | Event Handling | onClick/onChange/onSubmit, synthetic events, passing handlers, preventDefault |
| 06 | Conditional Rendering | `&&`, ternary, early return, switch with object map, rendering null |
| 07 | Lists and Keys | `.map()`, the `key` prop, why keys matter, key anti-patterns (index keys) |
| 08 | Forms and Inputs | Controlled vs uncontrolled, value/onChange, multi-field forms, FormData |
| 09 | Styling | Inline styles, CSS modules, Tailwind, CSS-in-JS overview, conditional classes |
| 10 | useEffect Basics | Side effects, dependency arrays, cleanup, common mistakes |

### Intermediate (02-Intermediate) — 20 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | useEffect Deep Dive | Stale closures, race conditions, AbortController, effect identity, when NOT to use |
| 02 | useRef and forwardRef | useRef for DOM/values, forwardRef, useImperativeHandle, callback refs |
| 03 | useMemo and useCallback | Memoization, referential equality, when not to use, profiling first |
| 04 | useReducer | Reducer pattern, action types, when to prefer over useState, with TypeScript |
| 05 | useContext | Context API, providers, consumers, performance pitfalls, splitting contexts |
| 06 | Custom Hooks | Naming (`useX`), rules of hooks, composition, returning tuple vs object |
| 07 | Component Composition | Children, slots, compound components, render props (legacy), prop drilling |
| 08 | Higher-Order Components | HOC pattern, when hooks replace HOCs, when HOCs still help (typing wrappers) |
| 09 | Forms Advanced | React Hook Form, Zod schema validation, controlled vs uncontrolled tradeoffs |
| 10 | Fetching Data | fetch + useEffect patterns, loading/error states, abort on unmount, race fixes |
| 11 | TanStack Query | useQuery, useMutation, query keys, caching, invalidation, devtools |
| 12 | State Management | Local vs lifted vs global, Context vs Zustand vs Redux Toolkit; pick chart |
| 13 | React Router | v6+ routes, nested routing, params, loaders/actions (data router), navigation |
| 14 | Code Splitting | React.lazy + Suspense, route-based splitting, dynamic import, prefetch hints |
| 15 | Error Boundaries | Class-only API, react-error-boundary, fallback UI, reset, logging |
| 16 | Portals | createPortal, modal/tooltip/toast patterns, event bubbling through portals |
| 17 | Refs and DOM | useRef vs useState, measuring, focus management, scroll, third-party DOM libs |
| 18 | Testing | Vitest + React Testing Library, user-event, queries (getBy/findBy), MSW |
| 19 | TypeScript with React | Component/prop types, generics, hooks typing, event types, polymorphic components |
| 20 | Accessibility | Semantic HTML, ARIA basics, focus management, keyboard nav, axe-core, reduced motion |

### Advanced (03-Advanced) — 18 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | React Internals | Reconciliation, fiber, render vs commit, double-invoke in StrictMode |
| 02 | Concurrent Features | useTransition, useDeferredValue, startTransition, interruptible rendering |
| 03 | Suspense | Suspense for lazy, for data, fallbacks, streaming, error+loading boundaries |
| 04 | Server Components | RSC mental model, server vs client boundary, `"use client"`, props serialization |
| 05 | Server Actions | React 19 Actions, `useActionState`, `useFormStatus`, progressive enhancement |
| 06 | Performance Optimization | React Profiler, memoization strategies, list virtualization (react-window/virtual) |
| 07 | Rendering Strategies | CSR vs SSR vs SSG vs ISR vs RSC; hydration, hydration mismatches |
| 08 | Next.js App Router | Layouts, loading.tsx, error.tsx, route groups, parallel/intercepting routes |
| 09 | Remix and React Router 7 | Loaders, actions, nested data routes, framework convergence |
| 10 | State Management Advanced | Redux Toolkit, RTK Query, Zustand, Jotai, signals (Preact/Solid comparison) |
| 11 | Forms at Scale | React Hook Form + Zod + Server Actions, multi-step wizards, optimistic UI |
| 12 | Animations | Framer Motion (motion/react), layout animations, exit animations, gestures |
| 13 | Internationalization | i18next, react-intl, ICU MessageFormat, locale-aware formatting, RTL |
| 14 | Design Systems | Headless UI primitives (Radix, Headless UI), shadcn/ui, tokens, theming |
| 15 | Micro-Frontends | Module federation (Webpack/rspack), single-spa, runtime composition tradeoffs |
| 16 | Build and Bundling | Vite, esbuild, Rollup, tree-shaking, bundle analysis, source maps |
| 17 | Monitoring and Errors | Sentry, web-vitals, RUM, error boundaries + logging, replay |
| 18 | React Native and Cross-Platform | RN core model, Expo, sharing logic between web and native, when not to share |

**Total: 48 notes** (10 beginner + 20 intermediate + 18 advanced)

---

## Diagram Coverage Targets

These notes specifically benefit from diagrams and **must** include one:

| Note | Diagram type |
|------|--------------|
| React Overview | graph (component tree → virtual DOM → real DOM) |
| Components and Props | classDiagram (component hierarchy with props flow) |
| State and useState | sequenceDiagram (setState → re-render cycle) |
| useEffect Basics | flowchart (mount / update / unmount lifecycle) |
| useEffect Deep Dive | sequenceDiagram (race condition + AbortController fix) |
| useReducer | flowchart (dispatch → reducer → new state → render) |
| useContext | graph (provider tree + consumer subscription) |
| Custom Hooks | graph (hook composition) |
| Component Composition | graph (children vs render props vs HOC) |
| Fetching Data | sequenceDiagram (component → fetch → state) |
| TanStack Query | flowchart (query lifecycle: fresh → stale → fetching → cached) |
| State Management | flowchart (decision tree: which state tool to pick) |
| React Router | graph (nested route tree) |
| Code Splitting | sequenceDiagram (route change → lazy chunk → render) |
| Error Boundaries | flowchart (error propagation + boundary catch) |
| Testing | flowchart (render → query → user event → assert) |
| React Internals | flowchart (render phase → commit phase) |
| Concurrent Features | sequenceDiagram (urgent vs transition update) |
| Suspense | sequenceDiagram (fallback → resolve → render) |
| Server Components | graph (server tree + client islands) |
| Server Actions | sequenceDiagram (form submit → action → revalidate) |
| Rendering Strategies | graph (CSR vs SSR vs SSG vs ISR vs RSC matrix) |
| Next.js App Router | graph (layout/page/loading/error file tree) |
| Design Systems | graph (tokens → primitives → patterns → pages) |
| Micro-Frontends | graph (host + remotes via module federation) |
| Build and Bundling | flowchart (source → transform → bundle → output) |
