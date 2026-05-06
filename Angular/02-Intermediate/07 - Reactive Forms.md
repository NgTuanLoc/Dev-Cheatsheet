---
tags: [angular, intermediate, forms]
aliases: [FormControl, FormGroup, FormBuilder, Typed Forms]
level: Intermediate
---

# Reactive Forms

> **One-liner**: Reactive forms build the form model **in code** with `FormGroup` / `FormControl` / `FormArray` — they're typed, programmatic, and the right choice for anything beyond a trivial form.

---

## Quick Reference

| API | Purpose |
|-----|---------|
| `ReactiveFormsModule` | Import in standalone component |
| `FormBuilder` | Convenience factory: `fb.group({...})` |
| `FormControl<T>(initial, validators?)` | Single field |
| `FormGroup<...>` | Object of controls |
| `FormArray<...>` | Dynamic list of controls |
| Validators | `Validators.required`, `email`, `minLength(n)`, `min(n)`, `pattern(re)` |
| Async validators | `(ctl) => Observable<ValidationErrors \| null>` |
| `valueChanges` / `statusChanges` | Observables of value / status |
| `getRawValue()` | Includes disabled controls |
| `patchValue` / `setValue` | Update partial / full |
| `.reset(value?)` | Mark pristine + untouched, optionally set value |

---

## Core Concept

Reactive forms move the form model out of the template into your TypeScript class. You declare a tree of controls explicitly with `FormBuilder`:

```ts
const form = fb.group({
  email: ['', [Validators.required, Validators.email]],
  password: ['', Validators.required],
});
```

The template binds to it via `[formGroup]` and `formControlName="..."`. Because you control the tree, you can **add/remove fields dynamically**, run **async validators** (server-side uniqueness check), do **cross-field validation** (password === confirmPassword), and react to changes with **`valueChanges`** as an Observable — exactly what you need for live filtering, debounced auto-save, conditional fields.

Since v14, forms are **typed** (`FormGroup<{ email: FormControl<string> }>`) — TS infers the shape from the initial value. Use `nonNullable: true` (or `fb.nonNullable.group(...)`) to avoid `T | null` everywhere.

For the "small login form" case, template-driven still wins on ergonomics. For everything else, reach for reactive forms.

---

## Diagram

```mermaid
classDiagram
    class FormGroup {
        +controls: { [k]: AbstractControl }
        +value
        +valid
        +valueChanges Observable
    }
    class FormControl {
        +value
        +valid
        +errors
    }
    class FormArray {
        +controls: AbstractControl[]
        +push, removeAt
    }
    FormGroup --> FormControl : has
    FormGroup --> FormArray : has
    FormArray --> FormControl : has
```

---

## Syntax & API

### A typed form

```ts
import { Component, inject } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

@Component({
  selector: 'app-signup',
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="submit()">
      <input formControlName="email" placeholder="Email" />
      @if (form.controls.email.invalid && form.controls.email.touched) {
        <p class="err">Invalid email</p>
      }

      <input formControlName="password" type="password" placeholder="Password" />

      <button [disabled]="form.invalid">Sign up</button>
    </form>
  `,
})
export class SignupComponent {
  private fb = inject(FormBuilder);

  form = this.fb.nonNullable.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
  });

  submit() {
    if (this.form.invalid) return;
    const { email, password } = this.form.getRawValue();
    console.log(email, password);
  }
}
```

### Nested groups

```ts
form = this.fb.nonNullable.group({
  profile: this.fb.nonNullable.group({
    firstName: ['', Validators.required],
    lastName:  ['', Validators.required],
  }),
  email: ['', [Validators.required, Validators.email]],
});
```

```html
<form [formGroup]="form">
  <div formGroupName="profile">
    <input formControlName="firstName" />
    <input formControlName="lastName" />
  </div>
  <input formControlName="email" />
</form>
```

### Dynamic `FormArray`

```ts
import { FormArray, FormControl } from '@angular/forms';

form = this.fb.nonNullable.group({
  tags: this.fb.array<FormControl<string>>([]),
});

addTag() { this.form.controls.tags.push(new FormControl('', { nonNullable: true })); }
removeTag(i: number) { this.form.controls.tags.removeAt(i); }
```

```html
<div formArrayName="tags">
  @for (ctl of form.controls.tags.controls; let i = $index; track i) {
    <input [formControlName]="i" />
    <button (click)="removeTag(i)">×</button>
  }
</div>
<button (click)="addTag()">Add tag</button>
```

### `valueChanges` for live updates

```ts
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { debounceTime, distinctUntilChanged } from 'rxjs';

constructor() {
  this.form.controls.search.valueChanges
    .pipe(debounceTime(250), distinctUntilChanged(), takeUntilDestroyed())
    .subscribe(q => this.api.search(q));
}
```

### Cross-field validator

```ts
import { AbstractControl, ValidationErrors } from '@angular/forms';

const passwordsMatch = (g: AbstractControl): ValidationErrors | null => {
  const pwd = g.get('password')?.value;
  const cfm = g.get('confirm')?.value;
  return pwd === cfm ? null : { mismatch: true };
};

form = this.fb.nonNullable.group({
  password: ['', Validators.required],
  confirm:  ['', Validators.required],
}, { validators: passwordsMatch });
```

---

## Common Patterns

```ts
// Pattern: patchValue from server data
load(id: number) {
  this.api.get(id).subscribe(user => this.form.patchValue(user));
}
```

```ts
// Pattern: signal-friendly current value via toSignal(form.valueChanges)
import { toSignal } from '@angular/core/rxjs-interop';

formValue = toSignal(this.form.valueChanges, { initialValue: this.form.getRawValue() });
isValid = computed(() => this.form.valid);
```

```ts
// Pattern: disabled when offline
constructor() {
  effect(() => this.online() ? this.form.enable() : this.form.disable());
}
```

---

## Gotchas & Tips

- **Use `nonNullable` (or `fb.nonNullable.group`)** — the default makes every control `T | null`, which infects the entire app.
- **`form.value` skips disabled controls.** Use `getRawValue()` if you need them too.
- **Setting a value with `setValue` requires every field**; `patchValue` accepts partial. Use `patchValue` for server sync.
- **`form.reset()` doesn't reset to the defaults you typed in the builder** — it resets to `null` (or the value you pass). Pass the original defaults explicitly.
- **`valueChanges` emits *after* a `set/patch`** — if you trigger a side effect from it, watch for loops. Use `emitEvent: false` to opt out.
- **Validators run for every value change.** Async validators (e.g. server uniqueness) should debounce inside the validator itself, not on `valueChanges`.
- **`formControlName` vs `[formControl]`**: use `formControlName` inside a `[formGroup]`; use `[formControl]="ctl"` for a standalone control.

---

## See Also

- [[06 - Template-Driven Forms]]
- [[08 - Custom Form Controls]]
- [[18 - Forms Validation Patterns]]
- [[12 - Forms at Scale]]
