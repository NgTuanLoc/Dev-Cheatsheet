---
tags: [angular, intermediate, routing]
aliases: [Angular Router, provideRouter, RouterOutlet]
level: Intermediate
---

# Routing Basics

> **One-liner**: Angular Router maps URLs to components — you declare `Routes`, register them with `provideRouter()`, drop a `<router-outlet />` where the matched component should render, and link with `[routerLink]`.

---

## Quick Reference

| API | Purpose |
|-----|---------|
| `Routes` | `{ path, component, loadComponent, children, ...}[]` |
| `provideRouter(routes)` | Bootstrap providers entry |
| `<router-outlet />` | Where the active route renders |
| `[routerLink]="['/x', id]"` | Declarative navigation |
| `routerLinkActive="active"` | Active-class binding |
| `Router.navigate(...)` | Imperative navigation |
| `ActivatedRoute` | Inject in component to read params |
| Params | `:id` in path → `route.paramMap.get('id')` |
| Query | `route.queryParamMap` |
| Wildcard | `path: '**'` for 404 |
| Redirect | `{ path: '', redirectTo: '/home', pathMatch: 'full' }` |

---

## Core Concept

The router is a **tree of routes** that matches against the URL. When you navigate (link click, programmatic, browser back), the router walks the tree, finds the deepest matching path, and renders the matched component into the nearest `<router-outlet />`.

Routes can be **nested** — a parent component has its own outlet that hosts its children's components. This is how layout shells work (a sidebar layout with feature pages inside).

You read URL **params** (`/users/:id`), **query params** (`?q=foo&page=2`), and **data** (static config attached to a route) by injecting `ActivatedRoute`. All three expose Observables and Maps.

For modern apps:

- **`provideRouter(routes)`** in bootstrap (no `RouterModule.forRoot` anymore).
- **Standalone components** with `loadComponent` for lazy routes.
- **Functional guards/resolvers** (see [[10 - Route Guards and Resolvers]]).

---

## Diagram

```mermaid
graph TD
    Root[/] --> Home[/home]
    Root --> Users[/users]
    Users --> UserList[UsersListComponent]
    Users --> UserId[/users/:id]
    UserId --> UserDetail[UserDetailComponent]
    Root --> Admin[/admin lazy]
    Admin --> AdminShell[AdminShell]
    AdminShell --> AdminUsers[/admin/users]
    AdminShell --> AdminRoles[/admin/roles]
```

---

## Syntax & API

### Define routes

```ts
// app.routes.ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  { path: '', redirectTo: 'home', pathMatch: 'full' },
  { path: 'home', loadComponent: () => import('./home/home.component').then(m => m.HomeComponent) },
  { path: 'users', loadComponent: () => import('./users/users.component').then(m => m.UsersComponent) },
  { path: 'users/:id', loadComponent: () => import('./users/user-detail.component').then(m => m.UserDetailComponent) },
  { path: 'admin', loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes) },
  { path: '**', loadComponent: () => import('./not-found.component').then(m => m.NotFoundComponent) },
];
```

### Bootstrap

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideRouter, withComponentInputBinding } from '@angular/router';
import { AppComponent } from './app/app.component';
import { routes } from './app/app.routes';

bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes, withComponentInputBinding())],
});
```

### Root component with outlet + nav

```ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  template: `
    <nav>
      <a routerLink="/home" routerLinkActive="active">Home</a>
      <a routerLink="/users" routerLinkActive="active">Users</a>
    </nav>
    <main>
      <router-outlet />
    </main>
  `,
})
export class AppComponent {}
```

### Read route params

```ts
import { Component, inject, signal } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { toSignal } from '@angular/core/rxjs-interop';
import { map } from 'rxjs';

@Component({ /* ... */ template: `<p>User #{{ id() }}</p>` })
export class UserDetailComponent {
  private route = inject(ActivatedRoute);
  id = toSignal(this.route.paramMap.pipe(map(p => +p.get('id')!)), { initialValue: 0 });
}
```

### Component input binding (Angular 16+)

With `withComponentInputBinding()` enabled, route params and query params bind directly to component `@Input()`s:

```ts
// Route: { path: 'users/:id', component: UserDetailComponent }
@Component({ /* ... */ })
export class UserDetailComponent {
  @Input() id!: string;            // bound from :id
  @Input() tab?: string;           // bound from ?tab=...
}
```

### Imperative navigation

```ts
import { Router } from '@angular/router';

const router = inject(Router);
router.navigate(['/users', userId], { queryParams: { tab: 'profile' } });
router.navigateByUrl('/home');
```

### `[routerLink]` with absolute / relative paths

```html
<a [routerLink]="['/users', user.id]">{{ user.name }}</a>
<a [routerLink]="['edit']">Edit (relative)</a>
<a routerLink="../">Up one</a>
<a [routerLink]="['/users']" [queryParams]="{ q: 'ada' }">Search</a>
```

---

## Common Patterns

```ts
// Pattern: nested routes with a layout
export const routes: Routes = [
  {
    path: 'admin',
    loadComponent: () => import('./admin/shell.component').then(m => m.ShellComponent),
    children: [
      { path: '', redirectTo: 'users', pathMatch: 'full' },
      { path: 'users', loadComponent: () => import('./admin/users.component').then(m => m.UsersComponent) },
      { path: 'roles', loadComponent: () => import('./admin/roles.component').then(m => m.RolesComponent) },
    ],
  },
];
```

```html
<!-- shell.component.html -->
<aside><app-nav /></aside>
<section><router-outlet /></section>
```

```ts
// Pattern: paramMap as a stream feeding switchMap
this.user$ = this.route.paramMap.pipe(
  switchMap(p => this.api.get(+p.get('id')!)),
);
```

---

## Gotchas & Tips

- **Use `pathMatch: 'full'` on redirects** for the empty path — without it you'll redirect on every URL.
- **Order matters.** More specific paths first; the wildcard `**` last.
- **Wildcard 404 is not the same as URL not handled.** It catches anything not matched higher in the tree, including unknown nested children of a matched parent.
- **Component input binding** (`withComponentInputBinding()`) is the modern way to read params — saves a lot of boilerplate.
- **Don't subscribe to `route.params` directly without unsubscribing** — use `paramMap` (cached) and convert to signal with `toSignal` or use the `async` pipe.
- **Lazy routes need `loadComponent` (single)** or **`loadChildren` (route group)**. Both return Promises that resolve to the component / Routes array.
- **`routerLink` without slash is relative** to the current route. With `/` it's absolute.

---

## See Also

- [[10 - Route Guards and Resolvers]]
- [[11 - Lazy Loading]]
- [[08 - Modules and Standalone Components]]
- [[03 - RxJS Operators]]
