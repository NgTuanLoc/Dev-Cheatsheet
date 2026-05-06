---
tags: [angular, intermediate, di]
aliases: [Hierarchical DI, Providers, InjectionToken]
level: Intermediate
---

# Dependency Injection Deep Dive

> **One-liner**: Angular's DI is **hierarchical** — every component has its own injector that delegates to its parent's, and providers can be placed at the root, on a route, or on a component to control scope and lifetime.

---

## Quick Reference

| Provider form | Effect |
|---------------|--------|
| `providedIn: 'root'` | App-wide singleton (default for services) |
| `providedIn: 'platform'` | Shared across multiple Angular apps on the page |
| `providedIn: 'any'` | New instance per lazy module (rarely useful) |
| `providers: [X]` on `@Component` | One instance per component instance + descendants |
| `providers: [X]` on a `Route` | One instance for the route's lifetime |
| `viewProviders: [X]` | Visible to the component's view, NOT to projected content |
| `{ provide: T, useClass: U }` | Use class `U` when `T` is requested |
| `{ provide: T, useValue: v }` | Use a static value |
| `{ provide: T, useFactory: () => ..., deps: [] }` | Build dynamically |
| `{ provide: T, useExisting: U }` | Alias to another binding |
| `multi: true` | Append to an array of values for the token |
| `@Optional()` | Don't throw if not found |
| `@Self()` | Only look in this injector |
| `@SkipSelf()` | Skip this injector, start at parent |
| `@Host()` | Stop at the component host |

---

## Core Concept

When you `inject(Foo)`, Angular asks the current injector "do you have `Foo`?" If yes, it returns the cached instance. If no, the request bubbles up the **injector tree** until something can satisfy it. If nothing can, you get a runtime `NullInjectorError`.

The tree has roughly three layers:

1. **Root injector** — built from `providers` in `bootstrapApplication`. Holds singletons.
2. **Route injectors** — created by lazy routes that declare `providers: [...]`. Live as long as the route is active.
3. **Component injectors** — one per component instance. Created when the component is instantiated, destroyed when it's removed.

This hierarchy is how you create **scoped instances**: provide a service on a component, and each instance of that component gets its own copy. Provide on a route, and the whole feature shares one. Provide at root, and the entire app shares one.

The four DI modifier decorators (`@Optional`, `@Self`, `@SkipSelf`, `@Host`) let you fine-tune the search — useful when you write directives or services that need to interact with parent components in specific ways.

---

## Diagram

```mermaid
graph TD
    Platform[Platform Injector] --> Root[Root Injector]
    Root --> RouteAdmin[/admin Route Injector]
    Root --> RouteUsers[/users Route Injector]
    RouteAdmin --> Comp1[Component A Injector]
    RouteAdmin --> Comp2[Component B Injector]
    Comp1 --> Comp1Child[Child Component Injector]
```

---

## Syntax & API

### `providedIn: 'root'`

```ts
@Injectable({ providedIn: 'root' })
export class LoggerService { /* singleton, tree-shakable */ }
```

### Component-scoped service

```ts
@Component({
  selector: 'app-cart',
  standalone: true,
  providers: [CartStore], // new CartStore per <app-cart>
  template: `...`,
})
export class CartComponent {
  store = inject(CartStore);
}
```

### Route-scoped service

```ts
{
  path: 'admin',
  providers: [AdminLogger],   // singleton for the lifetime of /admin
  loadChildren: () => import('./admin/admin.routes').then(m => m.routes),
}
```

### `useClass` for swap

```ts
abstract class Logger {
  abstract log(msg: string): void;
}

class ConsoleLogger extends Logger {
  log(m: string) { console.log(m); }
}

class NoopLogger extends Logger {
  log() { /* nothing */ }
}

bootstrapApplication(AppComponent, {
  providers: [
    { provide: Logger, useClass: environment.production ? NoopLogger : ConsoleLogger },
  ],
});
```

### `InjectionToken<T>` for non-class values

```ts
import { InjectionToken } from '@angular/core';

export const APP_CONFIG = new InjectionToken<{ apiUrl: string; version: string }>('APP_CONFIG');

bootstrapApplication(AppComponent, {
  providers: [
    { provide: APP_CONFIG, useValue: { apiUrl: '/api', version: '1.0.0' } },
  ],
});

// Consume
@Injectable({ providedIn: 'root' })
export class ApiService {
  private cfg = inject(APP_CONFIG);
}
```

### Multi-providers (arrays of values)

```ts
export const VALIDATORS = new InjectionToken<ValidatorFn[]>('VALIDATORS');

[
  { provide: VALIDATORS, useValue: requiredValidator, multi: true },
  { provide: VALIDATORS, useValue: emailValidator, multi: true },
];

// Consumer gets ValidatorFn[]
const all = inject(VALIDATORS);
```

### DI modifiers

```ts
import { Optional, Self, SkipSelf, Host, inject } from '@angular/core';

constructor(
  @Optional() private logger: Logger | null,    // null if not provided
  @Self() private localService: LocalThing,     // only this injector
  @SkipSelf() private parentService: ParentThing, // skip this, start at parent
  @Host() private hostService: HostThing,        // stop at component host
) {}

// inject() form
const logger = inject(Logger, { optional: true });
const parent = inject(ParentThing, { skipSelf: true });
```

---

## Common Patterns

```ts
// Pattern: provide an abstract class to swap implementations
abstract class Storage { abstract get(k: string): string | null; abstract set(k: string, v: string): void; }
class LocalStorageImpl extends Storage { /* ... */ }
class InMemoryImpl extends Storage { /* for tests */ }

// In production providers
{ provide: Storage, useClass: LocalStorageImpl }
// In tests
TestBed.configureTestingModule({ providers: [{ provide: Storage, useClass: InMemoryImpl }] });
```

```ts
// Pattern: feature config token
export const FEATURE_FLAGS = new InjectionToken<Record<string, boolean>>('FEATURE_FLAGS');

bootstrapApplication(AppComponent, {
  providers: [{ provide: FEATURE_FLAGS, useFactory: () => fetchFlags() }],
});
```

```ts
// Pattern: ENVIRONMENT_INITIALIZER for app startup work
import { ENVIRONMENT_INITIALIZER, inject } from '@angular/core';

bootstrapApplication(AppComponent, {
  providers: [
    {
      provide: ENVIRONMENT_INITIALIZER,
      multi: true,
      useValue: () => inject(SessionService).restoreFromStorage(),
    },
  ],
});
```

---

## Gotchas & Tips

- **`providedIn: 'root'` is the default for new services.** It's tree-shakable: unused services drop from the bundle.
- **Component-scoped services aren't shared with siblings** of the same component. They're per-instance.
- **`viewProviders` vs `providers`**: `viewProviders` aren't visible to content projected via `<ng-content>`. Use `providers` if a child component (placed via projection) needs the service.
- **Don't provide the same token twice in the same injector.** The last one wins, but it's confusing. Move shared providers up the tree.
- **Functional `inject()` is cleaner than `@Optional() / @SkipSelf()` decorators.** Use `inject(Foo, { optional: true, skipSelf: true })`.
- **Circular DI** between services usually means your model is wrong. Extract a third service that both depend on.
- **`InjectionToken<T>` is the right way to provide non-class values** (config, base URLs, environment). Don't use class names for non-class data.

---

## See Also

- [[07 - Services and DI Basics]]
- [[11 - Lazy Loading]]
- [[01 - Angular Internals]]
