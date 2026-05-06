---
tags: [angular, intermediate, forms]
aliases: [ControlValueAccessor, CVA, Custom Inputs]
level: Intermediate
---

# Custom Form Controls

> **One-liner**: A custom form control implements **`ControlValueAccessor`** — Angular's contract for "how forms read from and write to my component" — so your component can be used with `formControlName`, `formControl`, or `[(ngModel)]`.

---

## Quick Reference

| Method | Purpose |
|--------|---------|
| `writeValue(value)` | Form → component (set the value) |
| `registerOnChange(fn)` | Component → form (call when value changes) |
| `registerOnTouched(fn)` | Component → form (call on blur) |
| `setDisabledState?(disabled)` | Form → component (toggle disabled) |
| Provider | `NG_VALUE_ACCESSOR` — `multi: true` provider, `useExisting: forwardRef(...)` |

---

## Core Concept

When you write `<input formControlName="email">`, Angular's built-in `DefaultValueAccessor` (or one specific to checkboxes, radios, selects) sits between the `FormControl` and the DOM `<input>`. For your *own* component to participate, you implement that same contract: `ControlValueAccessor`.

The contract is four methods:

- **`writeValue(v)`** — the form tells you what value to display.
- **`registerOnChange(fn)`** — you keep this callback and call it every time the value changes inside your component.
- **`registerOnTouched(fn)`** — you call this on blur (so the form knows the user has interacted).
- **`setDisabledState(b)`** — optional, for `formControl.disable()` support.

You then register your component with the `NG_VALUE_ACCESSOR` token. After that, your component works anywhere a normal `<input>` would in a form.

In Angular 19+ there's an experimental signal-based forms API (preview) — but `ControlValueAccessor` is still the stable, production way to ship a custom input.

---

## Syntax & API

### Minimal CVA: a star-rating input

```ts
import { Component, forwardRef, signal } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'app-rating',
  standalone: true,
  template: `
    @for (n of [1,2,3,4,5]; track n) {
      <button type="button" (click)="set(n)" (blur)="onTouched()"
              [class.on]="n <= value()">★</button>
    }
  `,
  styles: [`button{background:none;border:0;color:#ccc;font-size:1.5rem}.on{color:gold}`],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => RatingComponent),
      multi: true,
    },
  ],
})
export class RatingComponent implements ControlValueAccessor {
  value = signal(0);
  disabled = signal(false);

  private onChange: (v: number) => void = () => {};
  onTouched: () => void = () => {};

  writeValue(v: number) { this.value.set(v ?? 0); }
  registerOnChange(fn: (v: number) => void) { this.onChange = fn; }
  registerOnTouched(fn: () => void) { this.onTouched = fn; }
  setDisabledState(b: boolean) { this.disabled.set(b); }

  set(n: number) {
    if (this.disabled()) return;
    this.value.set(n);
    this.onChange(n);
  }
}
```

### Using it in a form

```ts
@Component({
  standalone: true,
  imports: [ReactiveFormsModule, RatingComponent],
  template: `
    <form [formGroup]="form">
      <app-rating formControlName="stars" />
    </form>
    <p>Stars: {{ form.controls.stars.value }}</p>
  `,
})
export class ReviewFormComponent {
  form = inject(FormBuilder).nonNullable.group({ stars: 0 });
}
```

### Composite control wrapping a built-in input

```ts
@Component({
  selector: 'app-money-input',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <span>$</span>
    <input type="number" [formControl]="ctl" (blur)="onTouched()" />
  `,
  providers: [
    { provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => MoneyInputComponent), multi: true },
  ],
})
export class MoneyInputComponent implements ControlValueAccessor {
  ctl = new FormControl<number>(0, { nonNullable: true });
  onTouched: () => void = () => {};

  constructor() {
    this.ctl.valueChanges
      .pipe(takeUntilDestroyed())
      .subscribe(v => this.onChange(v));
  }
  private onChange: (v: number) => void = () => {};

  writeValue(v: number)  { this.ctl.setValue(v ?? 0, { emitEvent: false }); }
  registerOnChange(fn: (v: number) => void) { this.onChange = fn; }
  registerOnTouched(fn: () => void) { this.onTouched = fn; }
  setDisabledState(b: boolean) { b ? this.ctl.disable({ emitEvent: false }) : this.ctl.enable({ emitEvent: false }); }
}
```

---

## Common Patterns

```ts
// Pattern: validators on the wrapping form, not duplicated inside the CVA
form = this.fb.nonNullable.group({
  rating: [0, [Validators.required, Validators.min(1), Validators.max(5)]],
});
```

```ts
// Pattern: combine NG_VALUE_ACCESSOR with NG_VALIDATORS for a self-validating control
@Component({
  selector: 'app-color-hex',
  providers: [
    { provide: NG_VALUE_ACCESSOR, useExisting: forwardRef(() => ColorHexComponent), multi: true },
    { provide: NG_VALIDATORS,    useExisting: forwardRef(() => ColorHexComponent), multi: true },
  ],
  /* ... */
})
export class ColorHexComponent implements ControlValueAccessor, Validator {
  validate(c: AbstractControl): ValidationErrors | null {
    return /^#[0-9a-f]{6}$/i.test(c.value) ? null : { hex: true };
  }
  // ... CVA methods
}
```

---

## Gotchas & Tips

- **Don't forget `multi: true`** on the `NG_VALUE_ACCESSOR` provider — it's a multi-token. Without it you'll silently override built-in CVAs.
- **`forwardRef(() => Comp)`** is required because the class isn't defined yet when the decorator's metadata is evaluated. Use it whenever the provider references its own class.
- **Use `{ emitEvent: false }`** when reflecting `writeValue` into an internal `FormControl` — otherwise you'll loop (`writeValue → setValue → valueChanges → onChange → form → writeValue → ...`).
- **Call `onTouched()` on blur**, not on every keystroke. Touched-state UX depends on it.
- **Disabled state should propagate** — when the parent calls `formControl.disable()`, your inner controls should disable too.
- **For one-off controls inside a parent component**, you don't need a CVA — just `[(ngModel)]` on a regular input or use a `FormControl` directly. CVAs are for *reusable* inputs.

---

## See Also

- [[07 - Reactive Forms]]
- [[18 - Forms Validation Patterns]]
- [[16 - Custom Directives]]
- [[12 - Forms at Scale]]
