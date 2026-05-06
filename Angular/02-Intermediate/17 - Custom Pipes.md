---
tags: [angular, intermediate, pipes]
aliases: [Pipe class, Pure Pipe, Impure Pipe]
level: Intermediate
---

# Custom Pipes

> **One-liner**: A pipe is a class with `@Pipe` and a single `transform(value, ...args)` method — pure pipes are cached per input reference, impure pipes re-run every CD tick.

---

## Quick Reference

| Concept | Syntax |
|---------|--------|
| Decorator | `@Pipe({ name: 'myPipe', standalone: true })` |
| Default purity | `pure: true` (re-run only on input change) |
| Impure | `pure: false` (re-run every CD; use sparingly) |
| Method | `transform(value: T, ...args): U` |
| Use in template | `{{ value \| myPipe:arg }}` |
| Inject services | `inject(Foo)` in constructor / field |

---

## Core Concept

A pipe is a **template-only function** wrapped in a class. The class is mostly ceremony — Angular needs the metadata to register it. The actual work is the `transform(value, ...args)` method.

The big choice is **pure vs impure**:

- **Pure** (default): the pipe re-runs only when one of the inputs changes by reference (or by primitive equality). Pure pipes are cached, so `{{ users | filter:'admin' }}` inside a `@for` is cheap — same input, same output.
- **Impure**: re-runs on every change-detection tick. Use only when the pipe needs to re-evaluate even when inputs are the same reference (e.g. it's reading a stream — that's why `async` is impure).

If you need to filter or sort an array, **don't write an impure pipe** — do it in a `computed` signal instead. Impure pipes were a common React-vs-Angular point of contention, and the modern guidance is "write a `computed`, render the result."

For pipes that depend on locale or i18n, you'll often inject `LOCALE_ID` and use the `Intl` API.

---

## Syntax & API

### A simple pure pipe

```ts
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({
  name: 'truncate',
  standalone: true,
})
export class TruncatePipe implements PipeTransform {
  transform(value: string, max = 50, suffix = '…'): string {
    return value.length <= max ? value : value.slice(0, max - suffix.length) + suffix;
  }
}
```

```html
<p>{{ longText | truncate:100 }}</p>
<p>{{ tweet | truncate:140:'... [more]' }}</p>
```

### Pipe that injects a service

```ts
import { Pipe, PipeTransform, inject } from '@angular/core';
import { I18nService } from './i18n.service';

@Pipe({ name: 't', standalone: true })
export class TranslatePipe implements PipeTransform {
  private i18n = inject(I18nService);
  transform(key: string, params?: Record<string, unknown>): string {
    return this.i18n.translate(key, params);
  }
}
```

### Locale-aware number pipe

```ts
import { Pipe, PipeTransform, inject, LOCALE_ID } from '@angular/core';

@Pipe({ name: 'compactNumber', standalone: true })
export class CompactNumberPipe implements PipeTransform {
  private locale = inject(LOCALE_ID);
  private fmt = new Intl.NumberFormat(this.locale, { notation: 'compact' });

  transform(value: number | null | undefined): string {
    return value == null ? '' : this.fmt.format(value);
  }
}
```

```html
<span>{{ subscribers | compactNumber }}</span>
<!-- 1.2K, 3.4M, etc. -->
```

### Generic pipe

```ts
@Pipe({ name: 'first', standalone: true })
export class FirstPipe implements PipeTransform {
  transform<T>(arr: readonly T[] | null | undefined, n = 1): T[] {
    return arr ? arr.slice(0, n) : [];
  }
}
```

### Impure pipe (when truly needed)

```ts
@Pipe({ name: 'fileSize', standalone: true })
export class FileSizePipe implements PipeTransform {
  transform(bytes: number): string { /* pure formatting */ return format(bytes); }
}
// ↑ pure! No reason to be impure.

// Counterexample — DON'T do this for filtering arrays:
// @Pipe({ name: 'filter', pure: false })
// export class FilterPipe implements PipeTransform {
//   transform(items: T[], predicate: (t: T) => boolean): T[] { return items.filter(predicate); }
// }
// → use a computed signal instead.
```

---

## Common Patterns

```ts
// Pattern: prefer computed for derived collections
@Component({ /* ... */ template: `<li *ngFor="let u of admins()">{{ u.name }}</li>` })
export class UsersComponent {
  users = input<User[]>([]);
  admins = computed(() => this.users().filter(u => u.role === 'admin'));
}
// No pipe needed — and OnPush plays well with this.
```

```ts
// Pattern: localized date pipe wrapping Intl
@Pipe({ name: 'localDate', standalone: true })
export class LocalDatePipe implements PipeTransform {
  private locale = inject(LOCALE_ID);
  private fmt = new Intl.DateTimeFormat(this.locale, { dateStyle: 'medium', timeStyle: 'short' });
  transform(d: Date | string | null | undefined): string {
    if (!d) return '';
    return this.fmt.format(typeof d === 'string' ? new Date(d) : d);
  }
}
```

---

## Gotchas & Tips

- **Default to pure.** If you think you need impure, write a `computed` instead.
- **Pure pipes share one instance per template binding** but are cached on input identity. Multiple `| myPipe` calls with the same input compute once.
- **Don't allocate on each call** — cache the `Intl.NumberFormat` instance as a field, not inside `transform`. Reusing formatters is much faster.
- **Pipes can take type parameters**, but the template type checker doesn't always infer them. Add explicit return types.
- **Standalone pipes must be imported** by every component that uses them — add them to `imports`.
- **Pipes with side effects are bugs.** A pipe should be referentially transparent: same inputs → same output, no globals touched.

---

## See Also

- [[06 - Pipes]]
- [[16 - Custom Directives]]
- [[01 - Signals]]
- [[13 - Internationalization]]
