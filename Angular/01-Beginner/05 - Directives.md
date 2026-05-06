---
tags: [angular, beginner, directives, templates]
aliases: [Structural Directives, Attribute Directives, Control Flow]
level: Beginner
---

# Directives

> **One-liner**: Directives change how a template is rendered — **structural** ones (`@if`, `@for`, `@switch`) add/remove DOM, **attribute** ones (`ngClass`, `ngStyle`) modify existing elements.

---

## Quick Reference

| Modern syntax | Replaces | Use for |
|---------------|----------|---------|
| `@if (expr) { ... } @else { ... }` | `*ngIf` / `*ngIf...else` | Conditional render |
| `@for (item of items; track item.id) { ... }` | `*ngFor` (now requires `trackBy`) | Loops |
| `@switch (expr) { @case (x) {} @default {} }` | `*ngSwitch` / `*ngSwitchCase` | Branching |
| `@defer { ... } @placeholder { ... }` | (new — v17+) | Lazy / deferred render |
| `@let name = expr;` | (new — v18+) | Local template variable |
| `[ngClass]` / `[class.x]` | — | Conditional classes |
| `[ngStyle]` / `[style.x]` | — | Conditional styles |
| `[ngTemplateOutlet]` | — | Render `ng-template` from a ref |

---

## Core Concept

Angular has two flavors of directives:

- **Structural directives** change the DOM tree — they add or remove elements based on a condition. The new control flow (`@if`, `@for`, `@switch`) replaced the legacy `*ngIf`/`*ngFor`/`*ngSwitch` syntax in v17. New code should use `@if` / `@for` / `@switch` exclusively.
- **Attribute directives** stay on a single element and change its appearance or behavior — `ngClass`, `ngStyle`, custom directives like a `tooltip` or `clickOutside`.

The new control flow is **built into the compiler** — no imports needed. Legacy `*ngIf`/`*ngFor` come from `CommonModule` (or its standalone subset like `NgIf`, `NgFor`). Modern apps shouldn't import these.

`@for` **requires a `track` expression** — the equivalent of React's `key`. It tells Angular how to identify items so it can reuse DOM nodes when the list changes.

---

## Syntax & API

### `@if` / `@else`

```html
@if (user(); as u) {
  <p>Hello, {{ u.name }}</p>
} @else if (loading()) {
  <p>Loading…</p>
} @else {
  <p>Please log in.</p>
}
```

The `as u` clause aliases a truthy value (great for narrowing nullable signals).

### `@for` (always with `track`)

```html
<ul>
  @for (item of items(); track item.id; let i = $index) {
    <li>{{ i + 1 }}. {{ item.name }}</li>
  } @empty {
    <li>No items.</li>
  }
</ul>
```

Implicit variables: `$index`, `$first`, `$last`, `$even`, `$odd`, `$count`.

### `@switch`

```html
@switch (status()) {
  @case ('idle')    { <p>Waiting…</p> }
  @case ('loading') { <app-spinner /> }
  @case ('error')   { <p class="error">{{ errorMessage() }}</p> }
  @default          { <ng-content /> }
}
```

### `@let` (Angular 18+)

```html
@let total = items().reduce((s, i) => s + i.price, 0);
<p>Subtotal: {{ total | currency }}</p>
```

### `@defer` (Angular 17+)

```html
@defer (on viewport) {
  <app-heavy-chart />
} @placeholder { <p>Scroll down…</p> }
  @loading (after 100ms) { <app-spinner /> }
  @error { <p>Failed to load.</p> }
```

Trigger options: `on idle`, `on viewport`, `on interaction`, `on hover`, `on timer(2s)`, `when expr()`.

### `ngClass` / `ngStyle`

```html
<div [ngClass]="{ active: isActive, disabled: isDisabled }">…</div>
<div [ngStyle]="{ color: textColor, 'font-size.px': fontSize }">…</div>

<!-- Equivalent shorthand -->
<div [class.active]="isActive" [class.disabled]="isDisabled">…</div>
<div [style.color]="textColor" [style.font-size.px]="fontSize">…</div>
```

### Legacy `*ngIf` / `*ngFor` (for reference)

```html
<!-- Old — still works but discouraged -->
<p *ngIf="user as u; else loginTpl">Hello, {{ u.name }}</p>
<ng-template #loginTpl><p>Please log in.</p></ng-template>

<li *ngFor="let item of items; trackBy: trackById">{{ item.name }}</li>
```

---

## Common Patterns

```html
<!-- Pattern: optimistic skeleton with @if -->
@if (data(); as d) {
  <app-detail [item]="d" />
} @else {
  <app-skeleton />
}

<!-- Pattern: switch on a discriminated-union view-state -->
@switch (state().kind) {
  @case ('success') { <app-list [items]="state().data" /> }
  @case ('error')   { <app-error [message]="state().message" /> }
  @case ('loading') { <app-spinner /> }
}

<!-- Pattern: defer below the fold -->
@defer (on viewport) {
  <app-comments [postId]="postId()" />
}
```

---

## Gotchas & Tips

- **Always provide `track` in `@for`.** Without it the compiler errors out. Use the stable id (`track item.id`); use `track $index` only for static, never-reordered lists.
- **Don't import `NgIf` / `NgFor` / `NgSwitch`** in new components. The new control flow is a compiler feature, not a directive — no imports needed.
- **`@if` narrows types** like `@if (user())` lets you use the value inside without `?` chaining if you alias with `as u`.
- **`@defer` requires standalone components** for the deferred template. Pre-fetch with `prefetch on idle` if you want the chunk fetched before the trigger fires.
- **`ngClass` and `[class]` can both be used; prefer `[class.x]="condition"`** for one or two classes — it's the simplest and most readable.
- **Structural directives can't stack on the same element** (`<li *ngIf="..." *ngFor="...">` is invalid). Wrap in `<ng-container>` or use the new flow which handles this naturally.

---

## See Also

- [[06 - Pipes]]
- [[14 - Content Projection]]
- [[16 - Custom Directives]]
- [[06 - Performance Optimization]]
