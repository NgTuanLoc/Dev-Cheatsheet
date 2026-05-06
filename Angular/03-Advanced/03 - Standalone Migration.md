---
tags: [angular, advanced, architecture]
aliases: [NgModule Migration, Standalone Migration, ng generate @angular/core:standalone]
level: Advanced
---

# Standalone Migration

> **One-liner**: The Angular team ships an automated migration that converts NgModule-based apps to standalone components in three passes — components first, then routes/bootstrap, then NgModule deletion.

---

## Quick Reference

| Step | Command |
|------|---------|
| Convert components/directives/pipes to standalone | `ng g @angular/core:standalone --mode=convert-to-standalone` |
| Replace `RouterModule.forRoot` / NgModule bootstrap | `ng g @angular/core:standalone --mode=migrate-routes-and-bootstrap` |
| Delete now-unused NgModules | `ng g @angular/core:standalone --mode=prune-ng-modules` |
| Per-component flag | `standalone: true` (default in v19+) |
| Bootstrap | `bootstrapApplication(App, { providers })` |
| Module → providers | `XxxModule` → `provideX()` (e.g. `provideHttpClient`) |

---

## Core Concept

Pre-v14 Angular forced you to declare every component, directive, and pipe in an `NgModule`. Modules were also where you registered routing, HTTP, animations — everything app-wide.

Standalone components changed that: a component can declare its own template dependencies via `imports: [...]`. App-wide setup moves from `XxxModule` to **`provideX()` functions** in the bootstrap providers.

The migration to standalone is **mechanical** — Angular ships a schematic that does it for you. The recommended sequence:

1. **Convert components**: every `@Component`/`@Directive`/`@Pipe` becomes `standalone: true`, with their NgModule's `declarations` items moved into `imports`.
2. **Migrate the bootstrap**: `platformBrowserDynamic().bootstrapModule(AppModule)` becomes `bootstrapApplication(AppComponent, { providers: [...] })`. `RouterModule.forRoot` becomes `provideRouter`. `HttpClientModule` becomes `provideHttpClient`. And so on.
3. **Prune NgModules**: with no declarations and no providers, the modules are dead code. Delete them.

You can run the migration on a *part* of the app — Angular happily mixes NgModule-based and standalone code. Convert leaf features first; the shell and root come later.

---

## Syntax & API

### Run the migration

```bash
# Step 1 — convert components/directives/pipes
ng g @angular/core:standalone

# Pick mode: 1) Convert all components, directives and pipes to standalone
# Pick scope: src/app or a sub-path

# Step 2 — same command, pick mode 2
ng g @angular/core:standalone

# Step 3 — same command, pick mode 3 (prune)
ng g @angular/core:standalone
```

### Bootstrap migration

Before:

```ts
// app.module.ts
@NgModule({
  declarations: [AppComponent, NavComponent],
  imports: [BrowserModule, BrowserAnimationsModule, HttpClientModule, RouterModule.forRoot(routes)],
  providers: [{ provide: API_BASE_URL, useValue: '/api' }],
  bootstrap: [AppComponent],
})
export class AppModule {}

// main.ts
platformBrowserDynamic().bootstrapModule(AppModule);
```

After:

```ts
// main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { provideAnimations } from '@angular/platform-browser/animations';
import { provideHttpClient } from '@angular/common/http';
import { provideRouter } from '@angular/router';

bootstrapApplication(AppComponent, {
  providers: [
    provideAnimations(),
    provideHttpClient(),
    provideRouter(routes),
    { provide: API_BASE_URL, useValue: '/api' },
  ],
});
```

### Component migration

Before:

```ts
// nav.module.ts
@NgModule({
  declarations: [NavComponent],
  imports: [CommonModule, RouterModule],
  exports: [NavComponent],
})
export class NavModule {}

// nav.component.ts
@Component({ selector: 'app-nav', templateUrl: './nav.component.html' })
export class NavComponent {}
```

After:

```ts
// nav.component.ts
@Component({
  selector: 'app-nav',
  standalone: true,
  imports: [CommonModule, RouterModule],     // moved from module
  templateUrl: './nav.component.html',
})
export class NavComponent {}

// nav.module.ts — deleted in step 3
```

### Lazy routes migration

Before:

```ts
const routes: Routes = [
  { path: 'admin', loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule) },
];
```

After:

```ts
const routes: Routes = [
  { path: 'admin', loadChildren: () => import('./admin/admin.routes').then(m => m.adminRoutes) },
];
// admin.routes.ts is a Routes array, with each route loadComponent'ing a standalone component.
```

### Common module → provider mapping

| NgModule import | Standalone provider |
|-----------------|---------------------|
| `BrowserModule` | (auto-included by `bootstrapApplication`) |
| `BrowserAnimationsModule` | `provideAnimations()` |
| `NoopAnimationsModule` | `provideNoopAnimations()` |
| `HttpClientModule` | `provideHttpClient(withFetch())` |
| `RouterModule.forRoot(routes)` | `provideRouter(routes)` |
| `RouterModule.forChild(routes)` | (no replacement — routes are in lazy file directly) |
| `StoreModule.forRoot(...)` (NgRx) | `provideStore({...})` |
| `EffectsModule.forRoot([])` | `provideEffects([])` |
| `MatXxxModule` (Material) | import the components directly |

---

## Common Patterns

```ts
// Pattern: hybrid app — old NgModule + new standalone routes
// Keep AppModule for legacy code, but bootstrap gets a standalone shell.
// In v15+, you can `importProvidersFrom(LegacyModule)` to wire in NgModule-based providers:
import { importProvidersFrom } from '@angular/core';

bootstrapApplication(AppComponent, {
  providers: [
    importProvidersFrom(LegacyModule),
    provideRouter(routes),
  ],
});
```

```ts
// Pattern: convert one feature at a time
// Pre-step: pick a leaf feature with no NgModule consumers.
// 1. Run schematic on `src/app/features/foo` only.
// 2. Verify build, tests pass.
// 3. Move up the tree.
```

---

## Gotchas & Tips

- **Run the migration in clean git state.** It rewrites many files; review the diff before committing.
- **`importProvidersFrom`** is your friend for legacy NgModules whose APIs aren't yet standalone. It pulls module providers into the standalone bootstrap.
- **Don't migrate route-level NgModules with lazy loading** by hand — change `loadChildren` to point at a `Routes` array file (`admin.routes.ts`) and convert at the route boundary.
- **`CommonModule` is a noisy import.** After the migration, replace it with the new control flow (`@if`, `@for`) and individual pipe/directive imports for tighter bundles.
- **Tests need updates too.** `TestBed.configureTestingModule({ imports: [MyStandaloneComp] })` instead of `declarations`.
- **The migration is reversible** in chunks — if step 2 produces a regression, you can revert and retry. Steps are designed to be incremental.

---

## See Also

- [[08 - Modules and Standalone Components]]
- [[20 - Angular CLI Workflow]]
- [[09 - Routing Basics]]
