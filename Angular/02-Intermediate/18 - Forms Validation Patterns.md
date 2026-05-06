---
tags: [angular, intermediate, forms]
aliases: [Validators, Async Validators, Cross-Field Validation]
level: Intermediate
---

# Forms Validation Patterns

> **One-liner**: Validation in Angular happens at three levels — single-field sync (`Validators.required`), single-field async (server uniqueness checks), and cross-field validation on a `FormGroup` — and the UI gates display on `dirty`/`touched`.

---

## Quick Reference

| Built-in | Use |
|----------|-----|
| `Validators.required` | Field must have a value |
| `Validators.minLength(n)` / `maxLength(n)` | String/array length |
| `Validators.min(n)` / `max(n)` | Numeric bounds |
| `Validators.email` | Email format |
| `Validators.pattern(re)` | Regex |
| `Validators.requiredTrue` | Checkbox must be checked |
| Custom sync | `(ctl) => ValidationErrors \| null` |
| Custom async | `(ctl) => Observable<ValidationErrors \| null>` |
| Cross-field | Apply to `FormGroup` via `validators` |
| When to show | `ctl.invalid && (ctl.dirty \|\| ctl.touched)` |

---

## Core Concept

A **validator** is a function: it takes a `FormControl` (or group/array) and returns an object of errors (or `null` for valid). Errors live on `control.errors` as a plain object (`{ required: true, minlength: { requiredLength: 8, actualLength: 3 } }`).

Validators are pure functions — no state, no side effects. **Async validators** wrap an Observable that emits the same `ValidationErrors | null`. While the async validator runs, `control.pending` is `true` — useful for showing spinners.

For **cross-field validation** (passwords match, end-date after start-date), apply a validator to the parent `FormGroup`. The error lives on the group, not on the individual fields, so display logic is slightly different.

The hardest part isn't writing validators — it's **showing errors at the right time**. You don't want to flash "Required" before the user has typed anything. The standard rule: show errors only when `invalid && (dirty || touched)` — the user has either typed in the field or moved focus away.

---

## Syntax & API

### Built-in validators

```ts
import { FormBuilder, Validators } from '@angular/forms';

form = inject(FormBuilder).nonNullable.group({
  email:    ['', [Validators.required, Validators.email]],
  password: ['', [Validators.required, Validators.minLength(8)]],
  age:      [0,  [Validators.min(13), Validators.max(120)]],
  agree:    [false, Validators.requiredTrue],
});
```

### Custom sync validator

```ts
import { AbstractControl, ValidationErrors, ValidatorFn } from '@angular/forms';

export const noProfanity: ValidatorFn = (c: AbstractControl): ValidationErrors | null => {
  const banned = ['badword1', 'badword2'];
  return banned.some(w => (c.value as string).includes(w))
    ? { profanity: true }
    : null;
};

// Usage
{ comment: ['', [Validators.required, noProfanity]] }
```

### Validator factory (parameterized)

```ts
export const minWords = (n: number): ValidatorFn => (c) => {
  const words = (c.value as string).trim().split(/\s+/).filter(Boolean);
  return words.length >= n ? null : { minWords: { required: n, actual: words.length } };
};

// Usage
{ bio: ['', [minWords(20)]] }
```

### Async validator (server uniqueness)

```ts
import { AsyncValidatorFn } from '@angular/forms';
import { map, catchError, of, debounceTime, take, switchMap } from 'rxjs';

export const usernameAvailable = (api: UsersApi): AsyncValidatorFn =>
  (c) => of(c.value).pipe(
    debounceTime(300),
    take(1),
    switchMap(value =>
      api.checkUsername(value).pipe(
        map(taken => taken ? { taken: true } : null),
        catchError(() => of(null)),       // be lenient on transient errors
      ),
    ),
  );

// Usage
form = this.fb.nonNullable.group({
  username: ['', {
    validators: [Validators.required, Validators.minLength(3)],
    asyncValidators: [usernameAvailable(inject(UsersApi))],
    updateOn: 'blur',                     // only check on blur
  }],
});
```

### Cross-field validator

```ts
const passwordsMatch: ValidatorFn = (g) => {
  const a = g.get('password')?.value;
  const b = g.get('confirm')?.value;
  return a === b ? null : { mismatch: true };
};

form = this.fb.nonNullable.group({
  password: ['', Validators.required],
  confirm:  ['', Validators.required],
}, { validators: passwordsMatch });
```

```html
@if (form.errors?.['mismatch'] && form.controls.confirm.touched) {
  <p class="err">Passwords don't match.</p>
}
```

---

## Common Patterns

```ts
// Pattern: error message helper
type ErrorMessages = Record<string, (e: any) => string>;

const messages: ErrorMessages = {
  required:  () => 'Required',
  email:     () => 'Invalid email',
  minlength: e => `Min ${e.requiredLength} characters`,
  min:       e => `Must be at least ${e.min}`,
  max:       e => `Must be at most ${e.max}`,
  taken:     () => 'Already taken',
  mismatch:  () => "Doesn't match",
  profanity: () => 'Please rephrase',
};

export function firstError(errors: ValidationErrors | null): string {
  if (!errors) return '';
  const [key, payload] = Object.entries(errors)[0];
  const fmt = messages[key];
  return fmt ? fmt(payload) : 'Invalid';
}
```

```html
<input formControlName="email" />
@if (form.controls.email.invalid && form.controls.email.touched) {
  <p class="err">{{ firstError(form.controls.email.errors) }}</p>
}
```

```ts
// Pattern: signal-based pending indicator
isPending = toSignal(this.form.statusChanges, { initialValue: this.form.status }).pipe;
// or
isPending = computed(() => this.form.status === 'PENDING');
```

---

## Gotchas & Tips

- **Async validators run on every value change by default** — they can spam the network. Set `updateOn: 'blur'` or use `debounceTime` inside the validator.
- **A validator must return `null` for valid**, not `undefined`. Returning `undefined` flags it as invalid.
- **Cross-field errors live on the group**, not the children. Display them at the group level.
- **`statusChanges` and `valueChanges` are Observables.** Use `toSignal` if you want to bind to them in OnPush templates without `async` pipe.
- **`Validators.email` is loose** (any `x@y` passes). For strict checks use a regex or rely on backend validation.
- **Don't validate format AND uniqueness in the same async validator.** Sync validators run first; if format is invalid, async won't fire — keep them separate.
- **Reset clears errors only with a value reset.** `form.reset()` resets `dirty`, `touched`, and value; for selective reset, set new values without touching `dirty` (`patchValue` + `markAsPristine`).

---

## See Also

- [[07 - Reactive Forms]]
- [[08 - Custom Form Controls]]
- [[12 - Forms at Scale]]
- [[03 - RxJS Operators]]
