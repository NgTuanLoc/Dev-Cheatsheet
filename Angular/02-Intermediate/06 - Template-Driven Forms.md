---
tags: [angular, intermediate, forms]
aliases: [ngModel, FormsModule, Template Forms]
level: Intermediate
---

# Template-Driven Forms

> **One-liner**: Template-driven forms put `[(ngModel)]` and validation directives directly in the template — Angular silently builds the `FormControl` tree behind the scenes; great for small, simple forms.

---

## Quick Reference

| Directive | Use |
|-----------|-----|
| `FormsModule` | Import in standalone component |
| `[(ngModel)]="x"` | Two-way bind input → field |
| `name="..."` | Required for `ngModel` inside `<form>` |
| `#x="ngModel"` | Get the FormControl reference |
| `#f="ngForm"` | Get the form reference |
| Validators | `required`, `minlength`, `maxlength`, `pattern`, `email`, `min`, `max` |
| State props | `pristine` / `dirty`, `untouched` / `touched`, `valid` / `invalid`, `errors` |

---

## Core Concept

Template-driven forms let you treat a `<form>` and its inputs as the source of truth — `[(ngModel)]` two-way-binds an input to a field on the component, and Angular wires up a `FormControl` you never see.

You add validation by sprinkling **directives** (`required`, `minlength="3"`, `email`, `pattern="..."`) on the inputs. Each control exposes its state (`valid`, `dirty`, `touched`) which you read in the template — usually via a template reference variable like `#name="ngModel"`.

This style shines for **small forms**: a login screen, a profile editor with five fields, a search bar. Once the form has dynamic fields, conditional validation, async validators, or multiple submit paths, switch to **reactive forms** ([[07 - Reactive Forms]]) — the imperative API scales much better.

The runtime cost is real: every `ngModel` runs change detection on each input event. For large forms with many fields, reactive forms perform better.

---

## Syntax & API

### Basic form

```ts
import { Component } from '@angular/core';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [FormsModule],
  template: `
    <form #f="ngForm" (ngSubmit)="submit(f)">
      <label>
        Email
        <input name="email" type="email" required email
               [(ngModel)]="email" #emailCtl="ngModel" />
      </label>
      @if (emailCtl.invalid && emailCtl.touched) {
        @if (emailCtl.errors?.['required']) { <p class="err">Required</p> }
        @if (emailCtl.errors?.['email'])    { <p class="err">Invalid email</p> }
      }

      <label>
        Password
        <input name="password" type="password" required minlength="6"
               [(ngModel)]="password" #pwdCtl="ngModel" />
      </label>
      @if (pwdCtl.errors?.['minlength'] && pwdCtl.touched) {
        <p class="err">At least 6 characters</p>
      }

      <button type="submit" [disabled]="f.invalid">Sign in</button>
    </form>
  `,
})
export class LoginComponent {
  email = '';
  password = '';

  submit(f: NgForm) {
    if (f.invalid) return;
    console.log({ email: this.email, password: this.password });
  }
}
```

### Reading the form value

```ts
import { NgForm } from '@angular/forms';

submit(f: NgForm) {
  console.log(f.value);   // { email: '...', password: '...' }
  console.log(f.valid);
  console.log(f.touched);
}
```

### Disabled state

```html
<input name="role" [(ngModel)]="role" [disabled]="!isAdmin" />
```

### `ngModelOptions` for debounced commits

```html
<input name="q" [(ngModel)]="query" [ngModelOptions]="{ updateOn: 'blur' }" />
```

`updateOn: 'blur' | 'submit' | 'change'` (default `change`).

### Standalone `ngModel` (without a `<form>`)

```html
<input [(ngModel)]="quickFilter" />
```

When there's no surrounding `<form>`, `name` is optional — the input becomes a one-off control.

---

## Common Patterns

```html
<!-- Pattern: error block component or template helper -->
<input name="email" required email [(ngModel)]="email" #emailCtl="ngModel" />
<app-field-errors [control]="emailCtl" />
```

```html
<!-- Pattern: dirty + touched UX (only show errors after user interaction) -->
@if (ctl.invalid && (ctl.dirty || ctl.touched)) {
  <p class="err">{{ firstErrorMessage(ctl.errors) }}</p>
}
```

```ts
firstErrorMessage(errors: ValidationErrors | null): string {
  if (!errors) return '';
  if (errors['required'])  return 'Required';
  if (errors['email'])     return 'Invalid email';
  if (errors['minlength']) return `Min ${errors['minlength'].requiredLength} chars`;
  return 'Invalid';
}
```

---

## Gotchas & Tips

- **`name=""` is required** on every input inside a `<form>`. Without it, the control isn't registered and `f.value` won't include it.
- **Two-way `[(ngModel)]` runs CD on every keystroke.** For typeahead, switch to reactive forms with `valueChanges + debounceTime`, or use `ngModelOptions: { updateOn: 'blur' }`.
- **You can't easily add async validators** template-side. Reactive forms handle async cleanly.
- **Cross-field validation is awkward** — you end up writing custom directives. Use reactive forms.
- **`[disabled]` on a form control with `ngModel`** can produce a console warning. Use `[attr.disabled]` for non-form controls, or switch to reactive forms which support a `disabled` initial state.
- **Submit on Enter** works automatically inside `<form>` — `(ngSubmit)` wires up button + enter key.

---

## See Also

- [[07 - Reactive Forms]]
- [[08 - Custom Form Controls]]
- [[18 - Forms Validation Patterns]]
- [[12 - Forms at Scale]]
