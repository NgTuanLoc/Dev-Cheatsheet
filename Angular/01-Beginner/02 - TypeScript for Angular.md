---
tags: [angular, beginner, typescript]
aliases: [TypeScript Basics for Angular, TS for Angular]
level: Beginner
---

# TypeScript for Angular

> **One-liner**: Angular is written in TypeScript and assumes you'll write your code in TypeScript too — decorators, interfaces, generics, and strict mode are everyday tools, not optional ones.

---

## Quick Reference

| Feature | Use in Angular |
|---------|----------------|
| `class` | Components, services, directives, pipes |
| `@Decorator()` | `@Component`, `@Injectable`, `@Input`, `@Output`, `@Pipe`, `@Directive` |
| `interface` | Shape of API responses, props, configs |
| `type` | Unions, intersections, mapped types |
| `readonly` | Immutable fields (signals, configuration) |
| `?` | Optional properties / parameters |
| Generics `<T>` | `Observable<T>`, `Signal<T>`, custom services |
| `strict: true` | Always enabled in `tsconfig.json` |
| `Partial<T>` / `Pick<T,K>` / `Omit<T,K>` | Common utility types in form models |

---

## Core Concept

Angular leans on TypeScript heavily — **the framework's APIs are designed assuming you have static types.** Decorators (`@Component`, `@Injectable`) are TS metadata that the Angular compiler reads to wire components, providers, and templates. Without TypeScript you lose template type checking, DI inference, and most of the tooling.

You'll spend most of your time writing **classes** (for components, services, directives, pipes) and **interfaces** (for data shapes). Angular doesn't ask you to learn obscure type tricks — but a working knowledge of generics, utility types (`Partial`, `Pick`, `Omit`), and discriminated unions makes everyday code dramatically cleaner.

Always run with `"strict": true` in `tsconfig.json`. The Angular CLI sets this by default since v12. Strict mode catches null/undefined bugs at compile time and is the foundation for **strict template type checking**, which validates your `.html` files against component fields.

---

## Syntax & API

### Decorators

```ts
import { Component, Injectable, Input, Output, EventEmitter } from '@angular/core';

@Injectable({ providedIn: 'root' })
export class UserService {
  getCurrentUser() { /* ... */ }
}

@Component({ selector: 'app-greeting', standalone: true, template: `...` })
export class GreetingComponent {
  @Input() name = '';
  @Output() greeted = new EventEmitter<string>();
}
```

### Interfaces and types

```ts
export interface User {
  id: number;
  name: string;
  email: string;
  role: 'admin' | 'user';
  createdAt: Date;
}

export type UserCreate = Omit<User, 'id' | 'createdAt'>;
export type UserPatch = Partial<UserCreate>;
```

### Generics

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class ApiService {
  private http = inject(HttpClient);

  get<T>(url: string): Observable<T> {
    return this.http.get<T>(url);
  }
}

// Usage — fully typed
const users$ = api.get<User[]>('/api/users');
```

### `tsconfig.json` (Angular CLI defaults)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "preserve",
    "strict": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "experimentalDecorators": true,
    "useDefineForClassFields": false
  },
  "angularCompilerOptions": {
    "strictTemplates": true,
    "strictInputAccessModifiers": true
  }
}
```

---

## Common Patterns

```ts
// Pattern: discriminated union for view-state in templates
type ViewState<T> =
  | { kind: 'idle' }
  | { kind: 'loading' }
  | { kind: 'error'; message: string }
  | { kind: 'success'; data: T };

// In a component:
state = signal<ViewState<User[]>>({ kind: 'idle' });
```

```html
<!-- Template narrows on `kind` -->
@switch (state().kind) {
  @case ('loading') { <p>Loading…</p> }
  @case ('error')   { <p>Error: {{ state().message }}</p> }
  @case ('success') { <ul>@for (u of state().data; track u.id) { <li>{{ u.name }}</li> }</ul> }
}
```

---

## Gotchas & Tips

- **Always enable `strictTemplates`** in `angularCompilerOptions`. It type-checks `.html` files against your component class — catches typos and wrong types at compile time.
- **Decorators require `experimentalDecorators: true`** — the CLI sets this. Don't disable it.
- **Don't use `any`**. Reach for `unknown` if the type is genuinely unknown, then narrow with type guards.
- **Avoid `Function` and `Object`** — use proper signatures (`(arg: X) => Y`) and concrete types.
- **`readonly` your `inject()` results** to signal "this is a dependency, not state": `private readonly api = inject(ApiService)`.
- **Public API of a component**: `@Input()` / `@Output()` (or signal `input()` / `output()`) define the surface. Everything else should be `private` or `protected`.

---

## See Also

- [[01 - Angular Overview]]
- [[03 - Components and Templates]]
- [[07 - Services and DI Basics]]
- [[09 - Component Communication]]
