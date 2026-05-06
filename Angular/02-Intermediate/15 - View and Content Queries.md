---
tags: [angular, intermediate, templates]
aliases: [ViewChild, ContentChild, viewChild signal, ElementRef]
level: Intermediate
---

# View and Content Queries

> **One-liner**: Queries let a component reach into its own template (`viewChild` / `viewChildren`) or its projected content (`contentChild` / `contentChildren`) — modern Angular has signal-based `viewChild()` that's cleaner than the legacy `@ViewChild()` decorator.

---

## Quick Reference

| Query | What it returns | When |
|-------|-----------------|------|
| `viewChild(token)` | Signal of one match in own template | Reactive lookup |
| `viewChild.required(token)` | Required (non-null) signal | When you know it exists |
| `viewChildren(token)` | Signal of all matches in own template | List, reactive |
| `contentChild(token)` | Signal of one match in projected content | |
| `contentChildren(token)` | Signal of all matches in projected content | |
| `@ViewChild(token)` | Legacy decorator — set after `ngAfterViewInit` | |
| `@ViewChildren(token)` | Legacy QueryList | |
| `@ContentChild` / `@ContentChildren` | Legacy projected-content queries | |

---

## Core Concept

A **query** asks Angular "find me references inside my template" (or content). The result can be:

- A **template reference variable** (`<input #email>` queried by `'email'`).
- A **directive or component class** (queried by class).
- An **`ElementRef`** (queried with `{ read: ElementRef }`).
- A **`TemplateRef`** (a `<ng-template>` queried with `{ read: TemplateRef }`).

The signal-based queries (Angular 17.2+ stable) are far better than the legacy decorators:

- They're **signals** — read with `()` and they auto-track in `computed`/`effect`.
- They **re-evaluate when the DOM changes** — no fighting with "the child wasn't there at `ngOnInit`."
- `viewChild.required()` makes the signal non-null at the type level.

`viewChild` looks **only inside the component's own template**. `contentChild` looks **only at projected content** (what the parent put between the host tags). Don't confuse them.

---

## Syntax & API

### Signal-based view queries

```ts
import { Component, ElementRef, viewChild } from '@angular/core';

@Component({
  selector: 'app-canvas',
  standalone: true,
  template: `<canvas #c width="200" height="200"></canvas>`,
})
export class CanvasComponent {
  canvas = viewChild.required<ElementRef<HTMLCanvasElement>>('c');

  draw() {
    const ctx = this.canvas().nativeElement.getContext('2d')!;
    ctx.fillRect(10, 10, 100, 100);
  }
}
```

### Querying a child component

```ts
import { Component, viewChild } from '@angular/core';
import { ChildComponent } from './child.component';

@Component({
  standalone: true,
  imports: [ChildComponent],
  template: `<app-child /> <button (click)="ping()">Ping</button>`,
})
export class ParentComponent {
  child = viewChild.required(ChildComponent);
  ping() { this.child().sayHi(); }
}
```

### `viewChildren` (multiple)

```ts
import { Component, viewChildren } from '@angular/core';
import { TabComponent } from './tab.component';

@Component({ /* ... */ })
export class TabsComponent {
  tabs = viewChildren(TabComponent);

  closeAll() {
    this.tabs().forEach(t => t.close());
  }
}
```

### Content queries

```ts
import { Component, contentChildren } from '@angular/core';
import { TabComponent } from './tab.component';

@Component({
  selector: 'app-tabs',
  standalone: true,
  template: `
    <div class="tab-bar">
      @for (t of tabs(); track t) {
        <button (click)="t.activate()">{{ t.title() }}</button>
      }
    </div>
    <ng-content />
  `,
})
export class TabsComponent {
  tabs = contentChildren(TabComponent);
}
```

```html
<app-tabs>
  <app-tab title="One">…</app-tab>
  <app-tab title="Two">…</app-tab>
</app-tabs>
```

### Reading a `TemplateRef`

```ts
import { Component, contentChild, TemplateRef } from '@angular/core';

@Component({ /* ... */ })
export class ListComponent {
  rowTemplate = contentChild.required<TemplateRef<unknown>>('row');
}
```

```html
<app-list>
  <ng-template #row let-item>
    <strong>{{ item.name }}</strong>
  </ng-template>
</app-list>
```

### Legacy decorators (still common in older code)

```ts
import { Component, ViewChild, AfterViewInit } from '@angular/core';

@Component({ /* ... */ })
export class FooComponent implements AfterViewInit {
  @ViewChild('input') inputRef!: ElementRef<HTMLInputElement>;

  ngAfterViewInit() {
    this.inputRef.nativeElement.focus();
  }
}
```

---

## Common Patterns

```ts
// Pattern: focus on view init
@Component({ standalone: true, template: `<input #i />` })
export class FocusComponent {
  input = viewChild.required<ElementRef<HTMLInputElement>>('i');

  constructor() {
    effect(() => this.input().nativeElement.focus());
  }
}
```

```ts
// Pattern: ResizeObserver on a queried element
@Component({ /* ... */ })
export class ResizeAwareComponent {
  private el = viewChild.required<ElementRef<HTMLElement>>('host');
  width = signal(0);

  constructor() {
    effect((onCleanup) => {
      const node = this.el().nativeElement;
      const ro = new ResizeObserver(([e]) => this.width.set(e.contentRect.width));
      ro.observe(node);
      onCleanup(() => ro.disconnect());
    });
  }
}
```

---

## Gotchas & Tips

- **`viewChild()` returns a signal** — always read with `()`. Forgetting the parens is the #1 typo.
- **`viewChild.required()` throws if not found.** Use the non-required form if the element is conditional (`@if`-wrapped).
- **Content queries don't see content inside *child* components**, only direct projection. To query deeper, the child component should expose its own queries.
- **Decorator queries default `static: false`** (post-init). Pass `static: true` only if the queried element is unconditional and you need it inside `ngOnInit` — but signal queries don't need this distinction.
- **Don't query DOM you don't own.** If you want to react to a parent's child element, you're probably writing a custom directive — apply it to the child instead.
- **`{ read: ElementRef }`** is needed when the same selector matches both a directive and an element and you want the element.

---

## See Also

- [[14 - Content Projection]]
- [[10 - Lifecycle Hooks]]
- [[16 - Custom Directives]]
- [[01 - Signals]]
