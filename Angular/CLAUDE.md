# CLAUDE.md — Angular Topic

Topic-specific guidance for the `Angular/` section of the vault. For shared rules (template, Mermaid, authoring conventions), see the root [CLAUDE.md](../CLAUDE.md).

## What This Topic Is

A structured Angular teacher-authored cheatsheet/notebook — organized from beginner to advanced, intended for students learning modern Angular (17+ / 18+ / 19+) and the surrounding ecosystem (TypeScript, RxJS, Signals, Angular Router, NgRx, Angular Material).

---

## Version & Tooling Policy

- **Angular 17+ is the baseline**, with Angular 18 and 19 features (zoneless change detection, signal inputs/outputs, `@let`, deferrable views) marked explicitly when introduced.
- **Standalone components by default** — `NgModule` appears only as a brief historical note in [[01 - Angular Overview]] and [[08 - Modules and Standalone Components]]. Every example uses `standalone: true` (the default since v19).
- **New control flow (`@if` / `@for` / `@switch`)** is the default. Legacy structural directives (`*ngIf`, `*ngFor`, `*ngSwitch`) are mentioned only for compatibility.
- **Signals-first** — for new local state, prefer `signal()` / `computed()` / `effect()`. RxJS still owns async streams (HTTP, router events, websockets).
- **TypeScript-first** — all examples are TS. Plain JS is never shown.
- **Angular CLI (`ng`)** is the default tool. Examples assume `npm`/`pnpm` are available.
- **`inject()` over constructor injection** in most cases — cleaner with standalone APIs and required for some functional contexts (route guards, interceptors).

When a topic differs significantly across Angular versions, call it out:

````markdown
### Version notes

- **Angular 16**: Signals introduced (developer preview).
- **Angular 17**: New control flow, deferrable views, esbuild builder default.
- **Angular 18**: Material 3, event replay during hydration, signal-based forms preview.
- **Angular 19**: Standalone by default, signal `input()` / `output()` / `model()` stable, incremental hydration.
````

---

## Folder Structure

```
Angular/
├── CLAUDE.md                       ← this file
├── 00-Index/
│   ├── Angular Master Index.md     ← single entry point, links to all Angular notes
│   └── Angular Learning Path.md    ← recommended reading order per level
├── 01-Beginner/                    (10 notes)
├── 02-Intermediate/                (20 notes)
└── 03-Advanced/                    (18 notes)
```

---

## Tag Taxonomy (Angular-specific)

Always include `angular` plus the level. Add domain tags as relevant.

| Tag | Meaning |
|-----|---------|
| `angular` | applied to every Angular note |
| `typescript` | TypeScript-specific patterns (decorators, generics) |
| `templates` | template syntax, control flow, bindings |
| `directives` | structural and attribute directives |
| `pipes` | built-in and custom pipes |
| `di` | dependency injection, providers, injectors |
| `signals` | signal/computed/effect, signal-based APIs |
| `rxjs` | Observables, Subjects, operators |
| `forms` | template-driven and reactive forms |
| `routing` | Angular Router, guards, resolvers |
| `http` | HttpClient, interceptors, REST |
| `change-detection` | OnPush, zones, zoneless |
| `testing` | TestBed, Karma, Jest, Playwright |
| `cli` | `ng` CLI, schematics, builders |
| `ssr` | Angular Universal, hydration |
| `performance` | profiling, lazy loading, virtual scroll |
| `state-management` | NgRx, NGXS, signal stores, services |
| `architecture` | module/standalone design, folder layout |
| `accessibility` | a11y, CDK a11y |
| `styling` | component styles, ViewEncapsulation, Material |
| `animation` | `@angular/animations`, motion |
| `cdk` | Angular Component Dev Kit |
| `material` | Angular Material library |
| `monorepo` | Nx, workspaces |

---

## Code-Block Conventions

| Lang tag | Use for |
|----------|---------|
| `ts` | TypeScript class / service / standalone component file |
| `html` | Component template (`*.component.html`) |
| `scss` / `css` | Component styles |
| `bash` | CLI: `ng new`, `ng generate`, `npm install` |
| `json` | `angular.json`, `package.json`, `tsconfig.json` |
| `yaml` | CI configs (GitHub Actions) |
| `mermaid` | diagrams |

Examples must be **runnable** — student should be able to drop them into a fresh `ng new my-app --standalone` workspace and see them work. Prefer **inline templates** for short examples and **separate `.html`** for larger ones.

---

## Topic Coverage Map

### Beginner (01-Beginner) — 10 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Angular Overview | What Angular is, framework vs library, opinions, ecosystem map, CLI, standalone-first mental model |
| 02 | TypeScript for Angular | Decorators, classes, interfaces, generics, strict mode, why TS is mandatory |
| 03 | Components and Templates | `@Component`, selectors, inline vs external templates, ViewEncapsulation, host bindings |
| 04 | Data Binding | Interpolation `{{ }}`, property `[x]`, event `(x)`, two-way `[(x)]`, attribute vs property |
| 05 | Directives | Structural: `@if` / `@for` / `@switch` (and legacy `*ngIf`, `*ngFor`); attribute: `ngClass`, `ngStyle` |
| 06 | Pipes | Built-in pipes (`date`, `currency`, `async`, `json`), chaining, parameters, pure vs impure |
| 07 | Services and DI Basics | `@Injectable`, `providedIn: 'root'`, constructor injection vs `inject()`, why services exist |
| 08 | Modules and Standalone Components | NgModule legacy, standalone APIs, `imports`, bootstrapping with `bootstrapApplication` |
| 09 | Component Communication | `@Input()`, `@Output()`, `EventEmitter`, signal inputs/outputs (Angular 17.2+) |
| 10 | Lifecycle Hooks | `ngOnInit`, `ngOnChanges`, `ngOnDestroy`, `ngAfterViewInit`, when each fires |

### Intermediate (02-Intermediate) — 20 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Signals | `signal()`, `computed()`, `effect()`, glitch-free reads, when signals beat RxJS |
| 02 | RxJS Fundamentals | Observable vs Promise, `Subject`, `BehaviorSubject`, hot vs cold, subscription lifecycle |
| 03 | RxJS Operators | `map`, `filter`, `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`, `combineLatest`, `debounceTime` |
| 04 | HttpClient | `HttpClient`, typed responses, `withFetch`, headers, params, error handling |
| 05 | HTTP Interceptors | Functional interceptors, auth tokens, retry, error normalization, logging |
| 06 | Template-Driven Forms | `ngModel`, `FormsModule`, `NgForm`, when to use, validation messages |
| 07 | Reactive Forms | `FormControl`, `FormGroup`, `FormArray`, validators, async validators, typed forms |
| 08 | Custom Form Controls | `ControlValueAccessor`, integrating third-party inputs |
| 09 | Routing Basics | `provideRouter`, `Routes`, `routerLink`, `RouterOutlet`, route params, query params |
| 10 | Route Guards and Resolvers | Functional `CanActivate`, `CanMatch`, `CanDeactivate`, `Resolve`, `inject()` inside guards |
| 11 | Lazy Loading | Lazy routes, `loadComponent`, `loadChildren`, route-level providers |
| 12 | Dependency Injection Deep Dive | Hierarchical injectors, `provideX` patterns, multi providers, `@Optional`, `@Self`, `@SkipSelf` |
| 13 | Change Detection | Default vs `OnPush`, zones, `markForCheck`, when CD runs, signals + OnPush |
| 14 | Content Projection | `<ng-content>`, `select`, multi-slot projection, `ng-template`, `ng-container` |
| 15 | View and Content Queries | `@ViewChild`, `@ContentChild`, signal-based `viewChild()` / `contentChild()` |
| 16 | Custom Directives | Attribute directives, `HostBinding`, `HostListener`, structural directives with `TemplateRef` |
| 17 | Custom Pipes | `@Pipe`, transforming data, pure vs impure, signal-friendly pipes |
| 18 | Forms Validation Patterns | Sync + async validators, cross-field validation, displaying errors, dirty/touched UX |
| 19 | Testing Components | TestBed, `ComponentFixture`, `DebugElement`, change detection in tests, harnesses |
| 20 | Angular CLI Workflow | `ng generate`, `ng build`, `ng serve`, `ng test`, schematics, builders, workspace config |

### Advanced (03-Advanced) — 18 notes
| # | File | Topics Covered |
|---|------|----------------|
| 01 | Angular Internals | Ivy, view engine history, template compilation, CD tree, reactive primitives |
| 02 | Zone.js and Zoneless | What Zone.js does, `NgZone`, opting out with `provideExperimentalZonelessChangeDetection` |
| 03 | Standalone Migration | Migrating NgModule apps, schematic, hybrid apps, recommended layout |
| 04 | Server-Side Rendering | Angular Universal, `provideClientHydration`, SSR builders, transfer state |
| 05 | Hydration | Non-destructive hydration, event replay, incremental hydration (v19+), troubleshooting mismatches |
| 06 | Performance Optimization | Profiler, OnPush + signals, `trackBy`, virtual scroll, deferrable views (`@defer`) |
| 07 | NgRx and State Management | Store, actions, reducers, effects, `@ngrx/component-store`, signal store |
| 08 | RxJS Advanced | Higher-order observables, schedulers, multicasting, error recovery, custom operators |
| 09 | Animations | `@angular/animations`, triggers, states, transitions, route animations |
| 10 | Angular CDK | Overlay, portal, drag-drop, virtual scroll, a11y utilities |
| 11 | Angular Material | Theming, typography, components, density, customization with CSS vars |
| 12 | Forms at Scale | Reusable form patterns, dynamic forms, signal forms (preview), large-form perf |
| 13 | Internationalization | `@angular/localize`, ICU, locale-aware pipes, $localize |
| 14 | Build and Bundling | esbuild application builder, budgets, source maps, bundle analysis, lazy chunks |
| 15 | Micro-Frontends | Module federation with Angular, Native Federation, runtime composition |
| 16 | Monorepo with Nx | Nx workspace, generators, executors, affected commands, library boundaries |
| 17 | PWA and Service Worker | `@angular/pwa`, `ngsw-config.json`, caching strategies, app shell |
| 18 | Monitoring and Errors | `ErrorHandler`, Sentry, web-vitals, RUM, structured logging |

**Total: 48 notes** (10 beginner + 20 intermediate + 18 advanced)

---

## Diagram Coverage Targets

These notes specifically benefit from diagrams and **must** include one:

| Note | Diagram type |
|------|--------------|
| Angular Overview | graph (CLI → workspace → app → component tree) |
| Components and Templates | classDiagram (Component class + template + style) |
| Data Binding | flowchart (template ↔ class via 4 binding kinds) |
| Services and DI Basics | graph (component → injector → service) |
| Modules and Standalone Components | graph (NgModule tree vs standalone imports graph) |
| Component Communication | sequenceDiagram (parent → child via input, child → parent via output) |
| Lifecycle Hooks | stateDiagram-v2 (component lifecycle states) |
| Signals | flowchart (signal → computed → effect graph) |
| RxJS Fundamentals | sequenceDiagram (Observer subscribes, producer emits, completes) |
| HttpClient | sequenceDiagram (component → service → HttpClient → backend) |
| HTTP Interceptors | flowchart (request → interceptor chain → backend → response chain) |
| Reactive Forms | classDiagram (FormGroup composes FormControl + FormArray) |
| Routing Basics | graph (route tree with `RouterOutlet`s) |
| Route Guards and Resolvers | sequenceDiagram (navigation → guard → resolver → component) |
| Dependency Injection Deep Dive | graph (hierarchical injector tree) |
| Change Detection | flowchart (CD tick: root → children, OnPush gating) |
| Content Projection | graph (host → ng-content slots → projected content) |
| Custom Directives | sequenceDiagram (host element → directive lifecycle) |
| Testing Components | flowchart (TestBed → fixture → query → assert) |
| Angular CLI Workflow | flowchart (source → builder → bundle → output) |
| Angular Internals | flowchart (template → compile → view → CD) |
| Zone.js and Zoneless | sequenceDiagram (browser event → zone → CD vs zoneless signal-driven) |
| Server-Side Rendering | sequenceDiagram (request → render → HTML → hydrate) |
| Hydration | flowchart (server HTML → client bootstrap → claim DOM → activate) |
| Performance Optimization | flowchart (decision tree: which lever to pull) |
| NgRx and State Management | graph (component → action → reducer → store → selector → component) |
| Angular CDK | graph (overlay/portal/a11y modules + their primitives) |
| Build and Bundling | flowchart (TS + HTML + SCSS → esbuild → chunked bundles) |
| Micro-Frontends | graph (host shell + remote MFEs via module federation) |
| Monorepo with Nx | graph (apps + libs dependency graph) |
| PWA and Service Worker | sequenceDiagram (browser → SW → cache or network) |
