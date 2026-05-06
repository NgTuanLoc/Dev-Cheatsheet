---
tags: [angular, beginner, pipes, templates]
aliases: [Built-in Pipes, Template Pipes]
level: Beginner
---

# Pipes

> **One-liner**: Pipes are template-only functions that transform a value before it's displayed — `{{ value | pipeName:arg }}` — used for formatting dates, numbers, currency, and async values.

---

## Quick Reference

| Pipe | Example | Output |
|------|---------|--------|
| `date` | `{{ d \| date:'medium' }}` | `Aug 15, 2025, 3:42:00 PM` |
| `currency` | `{{ 1234.5 \| currency:'USD' }}` | `$1,234.50` |
| `number` | `{{ 3.14159 \| number:'1.0-2' }}` | `3.14` |
| `percent` | `{{ 0.42 \| percent }}` | `42%` |
| `uppercase` / `lowercase` | `{{ 'Hi' \| uppercase }}` | `HI` |
| `titlecase` | `{{ 'hello world' \| titlecase }}` | `Hello World` |
| `slice` | `{{ 'abcdef' \| slice:1:4 }}` | `bcd` |
| `json` | `{{ obj \| json }}` | `{ "x": 1 }` |
| `keyvalue` | `@for (kv of obj \| keyvalue; track kv.key)` | iterate object |
| `async` | `{{ user$ \| async }}` | unwrap Observable/Promise |

---

## Core Concept

A pipe is just a class with a `transform(value, ...args)` method, used in templates with the `|` syntax. They keep formatting code out of the component class — the class returns the *data*, the pipe formats it for display.

Pipes come in two flavors:

- **Pure** (default): re-runs only when the input reference changes. Pure pipes are cached, so calling them in a `@for` loop is cheap.
- **Impure**: re-runs on every change-detection tick (`@Pipe({ pure: false })`). Use sparingly — `async` is impure for a good reason (it watches subscriptions), but custom impure pipes are usually a code smell.

The **`async` pipe** is the most important one to know — it subscribes to an Observable or Promise, returns the latest value, and **automatically unsubscribes** when the component is destroyed. It's how you avoid manual `subscribe()` calls.

Standalone components must **import each pipe they use**. Most pipes live in `CommonModule`, but you can import them individually (`import { DatePipe } from '@angular/common'`).

---

## Syntax & API

### Importing pipes (standalone)

```ts
import { DatePipe, CurrencyPipe, AsyncPipe } from '@angular/common';

@Component({
  standalone: true,
  imports: [DatePipe, CurrencyPipe, AsyncPipe],
  template: `
    <p>{{ now | date:'short' }}</p>
    <p>{{ amount | currency:'USD' }}</p>
    <p>{{ user$ | async }}</p>
  `,
})
export class ReceiptComponent {
  now = new Date();
  amount = 49.99;
  user$ = this.api.getCurrentUser();
}
```

### Chaining pipes

```html
<p>{{ user.name | uppercase | slice:0:1 }}</p>
<p>{{ event.startsAt | date:'fullDate' | titlecase }}</p>
```

### Parameters

```html
<!-- date format strings -->
{{ d | date:'yyyy-MM-dd HH:mm' }}
{{ d | date:'short':'UTC':'en-US' }}

<!-- number digits info: minIntegerDigits.minFractionDigits-maxFractionDigits -->
{{ 3.14159 | number:'2.0-2' }}   <!-- 03.14 -->

<!-- currency: currencyCode, display, digitsInfo, locale -->
{{ price | currency:'EUR':'symbol':'1.2-2':'de' }}
```

### `async` pipe with Observables and signals

```ts
import { AsyncPipe } from '@angular/common';
import { toObservable } from '@angular/core/rxjs-interop';

@Component({
  standalone: true,
  imports: [AsyncPipe],
  template: `<p>{{ value$ | async }}</p>`,
})
export class FooComponent {
  count = signal(0);
  value$ = toObservable(this.count); // signal → Observable for the pipe
}
```

For pure signals you don't need `async` at all — just call the signal:

```html
<p>Count: {{ count() }}</p>
```

---

## Common Patterns

```html
<!-- Pattern: async pipe in @if to share the unwrapped value -->
@if (user$ | async; as user) {
  <p>Hello, {{ user.name }}</p>
  <p>Member since {{ user.createdAt | date:'mediumDate' }}</p>
}

<!-- Pattern: keyvalue iteration over a Map or plain object -->
@for (kv of settings | keyvalue; track kv.key) {
  <li>{{ kv.key }}: {{ kv.value }}</li>
}

<!-- Pattern: chaining for headers -->
<h2>{{ section.title | uppercase }}</h2>
```

---

## Gotchas & Tips

- **`async` is impure on purpose.** It re-checks the subscription each tick, which is exactly what you want. Don't write your own impure pipes unless you really know why.
- **Don't call methods inside `{{ }}` when a pipe will do.** `{{ formatPrice(p) }}` runs every CD; `{{ p | currency }}` is cached.
- **Pipes are component-scoped imports** in standalone code. Each component declares the pipes it uses in `imports`.
- **`json` pipe is for debugging** — pretty-prints any object. Drop one in during development to see why something isn't rendering.
- **`slice` returns a copy.** It's safe in `@for`, but two consecutive `| slice` expressions create two arrays — combine in one if perf matters.
- **Locale-aware pipes need the locale registered.** For non-English: `import '@angular/common/locales/global/de'; registerLocaleData(de);` or use `provideLocale()`.

---

## See Also

- [[05 - Directives]]
- [[17 - Custom Pipes]]
- [[01 - Signals]]
- [[02 - RxJS Fundamentals]]
