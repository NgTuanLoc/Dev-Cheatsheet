---
tags: [angular, intermediate, routing, performance]
aliases: [Lazy Routes, loadComponent, loadChildren]
level: Intermediate
---

# Lazy Loading

> **One-liner**: Lazy loading splits your bundle by route — `loadComponent` / `loadChildren` use dynamic `import()` so the chunk fetches the first time the user actually navigates there, keeping the initial bundle small.

---

## Quick Reference

| Use case | API |
|----------|-----|
| Single component per route | `loadComponent: () => import(...).then(m => m.X)` |
| Group of routes | `loadChildren: () => import(...).then(m => m.routes)` |
| Pre-load all lazy routes after bootstrap | `provideRouter(routes, withPreloading(PreloadAllModules))` |
| Selective preloading | Custom `PreloadingStrategy` |
| Defer template chunks | `@defer` block (see [[05 - Directives]]) |
| Route-level providers | `providers: [...]` on a `Route` |

---

## Core Concept

Modern bundlers (esbuild + Vite, Webpack) treat `import('./x')` as a **chunk boundary**. The lazy module ends up in its own JS file; the router fetches it the first time someone navigates to that path.

The two modes:

- **`loadComponent`** loads a single standalone component — perfect for "one route, one screen."
- **`loadChildren`** loads a `Routes` array (a feature module's worth of routes) — perfect for a whole feature area like `/admin`.

You can also **defer at the template level** with `@defer` blocks, which split a *part* of a single component (for example, hide a heavy chart below the fold). That's complementary to route-level lazy loading.

For UX, **preloading** matters: after the initial page renders and the network is idle, you can speculatively fetch lazy chunks so the next navigation feels instant. `PreloadAllModules` is fine for small apps; larger apps want a custom strategy that preloads only routes the user is likely to visit.

---

## Syntax & API

### Lazy single component

```ts
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'users',
    loadComponent: () => import('./users/users.component').then(m => m.UsersComponent),
  },
];
```

### Lazy route group

```ts
// app.routes.ts
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes),
}

// admin/admin.routes.ts
import { Routes } from '@angular/router';

export const adminRoutes: Routes = [
  {
    path: '',
    loadComponent: () => import('./shell/shell.component').then(m => m.ShellComponent),
    children: [
      { path: 'users', loadComponent: () => import('./users/users.component').then(m => m.UsersComponent) },
      { path: 'roles', loadComponent: () => import('./roles/roles.component').then(m => m.RolesComponent) },
    ],
  },
];
```

### Route-level providers (per-feature DI scope)

```ts
// admin/admin.routes.ts
import { provideState } from '@ngrx/store';
import { adminFeature } from './state';

export const adminRoutes: Routes = [
  {
    path: '',
    providers: [
      provideState(adminFeature),  // store slice only loaded with /admin
    ],
    loadComponent: () => import('./shell/shell.component').then(m => m.ShellComponent),
    children: [/* ... */],
  },
];
```

### Preloading

```ts
// main.ts
import { provideRouter, withPreloading, PreloadAllModules } from '@angular/router';

bootstrapApplication(AppComponent, {
  providers: [
    provideRouter(routes, withPreloading(PreloadAllModules)),
  ],
});
```

### Custom preloading strategy

```ts
import { PreloadingStrategy, Route } from '@angular/router';
import { Observable, of } from 'rxjs';

export class OnDataFlagStrategy implements PreloadingStrategy {
  preload(route: Route, load: () => Observable<unknown>): Observable<unknown> {
    return route.data?.['preload'] ? load() : of(null);
  }
}

// Mark routes
{ path: 'reports', data: { preload: true }, loadChildren: () => import('./reports/routes').then(m => m.routes) }

// Register
provideRouter(routes, withPreloading(OnDataFlagStrategy));
```

### `@defer` for sub-component lazy loading

```html
@defer (on viewport; prefetch on idle) {
  <app-comments [postId]="post.id" />
} @placeholder { <p>Scroll to see comments…</p> }
  @loading (after 100ms) { <app-spinner /> }
```

---

## Common Patterns

```ts
// Pattern: feature directory exports its own routes file
// Each feature owns: routes + components + services + state
features/
  admin/
    admin.routes.ts          // exports adminRoutes
    shell/                   // standalone components
    users/
    state/                   // feature state slice
```

```ts
// Pattern: route-level providers for feature-scoped services
{
  path: 'admin',
  providers: [
    AdminApiService,                 // singleton per /admin section, GC'd on leave
    { provide: API_BASE_URL, useValue: '/api/admin' },
  ],
  loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes),
}
```

---

## Gotchas & Tips

- **Lazy chunks need network on first visit.** Preload (or use `@defer`'s `prefetch on idle`) to mask the latency.
- **Route-level `providers` create a scope**: services provided here are singletons within the lazy area but garbage-collected when the user leaves it.
- **Don't `loadChildren` a single component.** Use `loadComponent` — simpler, smaller, faster.
- **Watch chunk size.** A 200KB lazy chunk that's loaded on every navigation is worse than a 20KB always-included one. Run `ng build --stats-json` and inspect.
- **Lazy + SSR**: lazy routes work in SSR; the server still bundles all routes. Use `@defer` for cases where you want the server NOT to render heavy content.
- **`loadChildren` returning a routes array** is the modern form. Anything returning `NgModule` is legacy and should be migrated.

---

## See Also

- [[09 - Routing Basics]]
- [[10 - Route Guards and Resolvers]]
- [[14 - Build and Bundling]]
- [[06 - Performance Optimization]]
