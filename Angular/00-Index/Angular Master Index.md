---
tags: [angular, index]
aliases: [Angular Index, Angular Home]
---

# Angular Master Index — Angular Cheatsheet

> **Single entry point** for the Angular topic. Browse by level or jump straight to a topic.

---

## Start Here

- [[Angular Learning Path]] — recommended reading order
- New to Angular? Begin with [[01 - Angular Overview]]

---

## 01 — Beginner (foundations)

The mental model + the everyday API. Master these before touching signals, RxJS, or routing.

| # | Note | What you'll learn |
|---|------|-------------------|
| 01 | [[01 - Angular Overview]] | What Angular is, framework vs library, ecosystem map |
| 02 | [[02 - TypeScript for Angular]] | Decorators, classes, interfaces, generics |
| 03 | [[03 - Components and Templates]] | `@Component`, selectors, templates, styles |
| 04 | [[04 - Data Binding]] | Interpolation, property, event, two-way binding |
| 05 | [[05 - Directives]] | `@if` / `@for` / `@switch`, `ngClass`, `ngStyle` |
| 06 | [[06 - Pipes]] | Built-in pipes, parameters, chaining |
| 07 | [[07 - Services and DI Basics]] | `@Injectable`, `inject()`, providers |
| 08 | [[08 - Modules and Standalone Components]] | Standalone-first, `bootstrapApplication`, legacy NgModule |
| 09 | [[09 - Component Communication]] | `@Input`, `@Output`, signal `input()` / `output()` |
| 10 | [[10 - Lifecycle Hooks]] | `ngOnInit`, `ngOnDestroy`, `ngAfterViewInit` |

---

## 02 — Intermediate (signals, RxJS, real apps)

Reactivity, HTTP, forms, routing, DI deep-dive, testing.

### Reactivity
| # | Note |
|---|------|
| 01 | [[01 - Signals]] |
| 02 | [[02 - RxJS Fundamentals]] |
| 03 | [[03 - RxJS Operators]] |

### HTTP
| # | Note |
|---|------|
| 04 | [[04 - HttpClient]] |
| 05 | [[05 - HTTP Interceptors]] |

### Forms
| # | Note |
|---|------|
| 06 | [[06 - Template-Driven Forms]] |
| 07 | [[07 - Reactive Forms]] |
| 08 | [[08 - Custom Form Controls]] |

### Routing
| # | Note |
|---|------|
| 09 | [[09 - Routing Basics]] |
| 10 | [[10 - Route Guards and Resolvers]] |
| 11 | [[11 - Lazy Loading]] |

### DI, change detection, projection
| # | Note |
|---|------|
| 12 | [[12 - Dependency Injection Deep Dive]] |
| 13 | [[13 - Change Detection]] |
| 14 | [[14 - Content Projection]] |
| 15 | [[15 - View and Content Queries]] |

### Authoring
| # | Note |
|---|------|
| 16 | [[16 - Custom Directives]] |
| 17 | [[17 - Custom Pipes]] |
| 18 | [[18 - Forms Validation Patterns]] |

### Quality & tooling
| # | Note |
|---|------|
| 19 | [[19 - Testing Components]] |
| 20 | [[20 - Angular CLI Workflow]] |

---

## 03 — Advanced (internals, performance, architecture)

Runtime, zoneless, SSR, hydration, NgRx, monorepos, deployment.

### Runtime & rendering
| # | Note |
|---|------|
| 01 | [[01 - Angular Internals]] |
| 02 | [[02 - Zone.js and Zoneless]] |
| 03 | [[03 - Standalone Migration]] |
| 04 | [[04 - Server-Side Rendering]] |
| 05 | [[05 - Hydration]] |

### Performance & state
| # | Note |
|---|------|
| 06 | [[06 - Performance Optimization]] |
| 07 | [[07 - NgRx and State Management]] |
| 08 | [[08 - RxJS Advanced]] |

### UI & UX
| # | Note |
|---|------|
| 09 | [[09 - Animations]] |
| 10 | [[10 - Angular CDK]] |
| 11 | [[11 - Angular Material]] |
| 12 | [[12 - Forms at Scale]] |
| 13 | [[13 - Internationalization]] |

### Architecture, build, monitoring
| # | Note |
|---|------|
| 14 | [[14 - Build and Bundling]] |
| 15 | [[15 - Micro-Frontends]] |
| 16 | [[16 - Monorepo with Nx]] |
| 17 | [[17 - PWA and Service Worker]] |
| 18 | [[18 - Monitoring and Errors]] |

---

## Browse by tag

Use Obsidian's tag pane to filter:

- `#signals` — signal/computed/effect, signal-based APIs
- `#rxjs` — Observables, operators, schedulers
- `#di` — Dependency injection, providers, injectors
- `#forms` — template-driven, reactive, validation
- `#routing` — Router, guards, resolvers, lazy loading
- `#http` — HttpClient, interceptors
- `#change-detection` — OnPush, zones, zoneless
- `#testing` — TestBed, harnesses
- `#ssr` — Universal, hydration
- `#performance` — profiling, deferrable views, virtual scroll
- `#state-management` — NgRx, signal stores, services
- `#cdk` `#material` — CDK + Material primitives
- `#cli` — Angular CLI, schematics, builders
- `#monorepo` — Nx workspaces
- `#architecture` — standalone, micro-frontends, layout
