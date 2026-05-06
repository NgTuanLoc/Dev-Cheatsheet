---
tags: [angular, index, learning-path]
aliases: [Angular Roadmap, Angular Curriculum]
---

# Learning Path

> **Recommended order** to study the vault. Stages assume you know HTML/CSS, basic JavaScript, and have read at least a chapter on TypeScript. Builds toward production-grade depth.

---

## Stage 1 — Angular Fundamentals (1 week)

**Goal**: Build a small interactive UI from scratch — components, templates, bindings, control flow.

1. [[01 - Angular Overview]] — orient the framework and CLI
2. [[02 - TypeScript for Angular]]
3. [[03 - Components and Templates]]
4. [[04 - Data Binding]]
5. [[05 - Directives]]
6. [[06 - Pipes]]
7. [[07 - Services and DI Basics]]
8. [[08 - Modules and Standalone Components]]
9. [[09 - Component Communication]]
10. [[10 - Lifecycle Hooks]]

**Checkpoint**: Build a TODO app with add/edit/delete, filter by status, persist via a service to `localStorage`.

---

## Stage 2 — Reactivity (1 week)

**Goal**: Choose the right tool — signals for synchronous derived state, RxJS for async streams. Most Angular bugs come from mixing them up.

1. [[01 - Signals]] ← **read first**
2. [[02 - RxJS Fundamentals]]
3. [[03 - RxJS Operators]]

**Checkpoint**: Refactor your TODO list to expose `signal`-based state and a `computed` for the filtered view; convert any subscriptions you had to declarative `async` pipe usage.

---

## Stage 3 — HTTP & Data (3–5 days)

**Goal**: Talk to a real backend with typed responses, interceptors, and good error handling.

1. [[04 - HttpClient]]
2. [[05 - HTTP Interceptors]]

**Checkpoint**: Replace the localStorage layer with a JSON REST API. Add a logging + auth interceptor.

---

## Stage 4 — Forms (1 week)

**Goal**: Ship validated forms — both template-driven (small) and reactive (everything else).

1. [[06 - Template-Driven Forms]]
2. [[07 - Reactive Forms]]
3. [[08 - Custom Form Controls]]
4. [[18 - Forms Validation Patterns]]

**Checkpoint**: Add a multi-field form with sync + async validation and a custom `ControlValueAccessor` (e.g. star-rating input).

---

## Stage 5 — Routing & Lazy Loading (3–5 days)

1. [[09 - Routing Basics]]
2. [[10 - Route Guards and Resolvers]]
3. [[11 - Lazy Loading]]

**Checkpoint**: Add nested routes, a route resolver, an auth guard, and lazy-load a feature area.

---

## Stage 6 — DI, Change Detection, Composition (1 week)

**Goal**: Understand what makes Angular tick. Most performance and "why won't it update?" bugs live here.

1. [[12 - Dependency Injection Deep Dive]]
2. [[13 - Change Detection]]
3. [[14 - Content Projection]]
4. [[15 - View and Content Queries]]
5. [[16 - Custom Directives]]
6. [[17 - Custom Pipes]]

**Checkpoint**: Convert your app to `OnPush` everywhere; identify and fix any places that break.

---

## Stage 7 — Quality (1 week)

1. [[19 - Testing Components]]
2. [[20 - Angular CLI Workflow]]

**Checkpoint**: 80% coverage on services, 60%+ on components, all green in CI.

---

## Stage 8 — Runtime & Server (1–2 weeks)

**Goal**: Understand how Angular renders, then push rendering to the server.

1. [[01 - Angular Internals]]
2. [[02 - Zone.js and Zoneless]]
3. [[06 - Performance Optimization]]
4. [[04 - Server-Side Rendering]]
5. [[05 - Hydration]]

**Checkpoint**: SSR your app, fix any hydration mismatches, measure TTFB and INP before/after.

---

## Stage 9 — State at Scale (1 week)

1. [[07 - NgRx and State Management]]
2. [[08 - RxJS Advanced]]
3. [[12 - Forms at Scale]]
4. [[03 - Standalone Migration]] *(skip if you started standalone)*

---

## Stage 10 — Design & UX (1 week)

1. [[10 - Angular CDK]]
2. [[11 - Angular Material]]
3. [[09 - Animations]]
4. [[13 - Internationalization]]

---

## Stage 11 — Architecture & Deployment (1 week)

1. [[14 - Build and Bundling]]
2. [[17 - PWA and Service Worker]]
3. [[18 - Monitoring and Errors]]
4. [[16 - Monorepo with Nx]]
5. [[15 - Micro-Frontends]]

---

## Tips for using this path

- **Don't skip Stage 2.** Confusing signals and RxJS — or subscribing without unsubscribing — is the top source of Angular bugs.
- **Build the checkpoint at each stage.** Reading without coding doesn't stick.
- **Don't reach for NgRx on day one.** A well-organized service with signals covers 80% of apps.
- **Use `OnPush` from the start** in any new component — it's free performance and surfaces CD bugs early.
- **Prefer standalone APIs.** NgModule is legacy in v17+ and removed-by-default thinking in v19+.
- Use [[Angular Master Index]] to jump around; this path is recommended, not mandatory.
