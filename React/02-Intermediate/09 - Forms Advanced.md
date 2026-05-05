---
tags: [react, intermediate, forms, typescript]
aliases: [React Hook Form, Zod, Form Validation]
level: Intermediate
---

# Forms Advanced

> **One-liner**: For real apps, use **React Hook Form** for form state (uncontrolled = fast, no per-keystroke re-renders) and **Zod** for schema-based validation that doubles as TypeScript types.

---

## Quick Reference

| Tool | Why |
|------|-----|
| **React Hook Form** | Uncontrolled by default → minimal re-renders, integrates with native validation |
| **Zod** | Schema-first validation, infers TS types from the schema |
| **`@hookform/resolvers/zod`** | Glue: pass a Zod schema to RHF |
| **Yup** | Older alternative to Zod |
| **Formik** | Older alternative to RHF, controlled-by-default — slower for large forms |
| **TanStack Form** | Newer competitor with better TS ergonomics |

---

## Core Concept

Manual controlled forms scale poorly: every keystroke triggers a re-render of the whole form, validation gets tangled with state, and TypeScript types drift from runtime checks.

**React Hook Form (RHF)** solves the perf problem by being **uncontrolled** — inputs register themselves with RHF, which reads values via refs, only re-rendering parts that need to (errors, dirty state). Submitting collects everything.

**Zod** solves the validation+typing problem: you describe your form schema once, and Zod gives you both runtime validation and a TypeScript type — eliminating drift.

The combo (`zodResolver`) is the 2025 default for serious React forms. For React 19, integrate with **Server Actions** for form submission ([[05 - Server Actions]]).

---

## Syntax & API

### Setup

```bash
npm install react-hook-form zod @hookform/resolvers
```

### Schema → types → form, end-to-end

```tsx
import { useForm } from "react-hook-form";
import { z } from "zod";
import { zodResolver } from "@hookform/resolvers/zod";

const schema = z.object({
  email:    z.string().email("Enter a valid email"),
  password: z.string().min(8, "At least 8 characters"),
  age:      z.coerce.number().int().min(13, "Must be 13+"),
  agree:    z.boolean().refine(v => v === true, "Required"),
});

type FormValues = z.infer<typeof schema>;     // ← TS type from schema

function SignUpForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting, isValid },
  } = useForm<FormValues>({
    resolver: zodResolver(schema),
    mode: "onBlur",                            // validate on blur (vs onChange/onSubmit)
    defaultValues: { email: "", password: "", age: 18, agree: false },
  });

  const onSubmit = handleSubmit(async (values) => {
    await api.signUp(values);
  });

  return (
    <form onSubmit={onSubmit}>
      <label>
        Email
        <input type="email" {...register("email")} />
        {errors.email && <p className="err">{errors.email.message}</p>}
      </label>

      <label>
        Password
        <input type="password" {...register("password")} />
        {errors.password && <p className="err">{errors.password.message}</p>}
      </label>

      <label>
        Age
        <input type="number" {...register("age")} />
        {errors.age && <p className="err">{errors.age.message}</p>}
      </label>

      <label>
        <input type="checkbox" {...register("agree")} /> I agree to terms
      </label>
      {errors.agree && <p className="err">{errors.agree.message}</p>}

      <button type="submit" disabled={isSubmitting || !isValid}>
        {isSubmitting ? "Saving…" : "Sign up"}
      </button>
    </form>
  );
}
```

### Controlled inputs inside RHF (for libraries that need them)

```tsx
import { Controller } from "react-hook-form";
import { Select } from "some-ui-library";

<Controller
  name="country"
  control={control}
  render={({ field, fieldState }) => (
    <>
      <Select {...field} options={countries} />
      {fieldState.error && <p>{fieldState.error.message}</p>}
    </>
  )}
/>
```

### Field arrays — dynamic lists of fields

```tsx
import { useFieldArray } from "react-hook-form";

const { fields, append, remove } = useFieldArray({ control, name: "items" });

{fields.map((field, idx) => (
  <div key={field.id}>
    <input {...register(`items.${idx}.name` as const)} />
    <input type="number" {...register(`items.${idx}.qty` as const)} />
    <button onClick={() => remove(idx)}>x</button>
  </div>
))}
<button onClick={() => append({ name: "", qty: 1 })}>+ add</button>
```

---

## Common Patterns

```tsx
// Pattern: server-side errors
const onSubmit = handleSubmit(async (values) => {
  try {
    await api.signUp(values);
  } catch (e) {
    if (e instanceof ApiError && e.field) {
      setError(e.field, { message: e.message });
    } else {
      setError("root", { message: "Something went wrong" });
    }
  }
});

{errors.root && <p className="err">{errors.root.message}</p>}
```

```tsx
// Pattern: cross-field validation with Zod
const schema = z.object({
  password: z.string().min(8),
  confirm:  z.string(),
}).refine(d => d.password === d.confirm, {
  message: "Passwords don't match",
  path: ["confirm"],
});
```

```tsx
// Pattern: progressive enhancement with React 19 actions + RHF
<form action={signUpAction} onSubmit={handleSubmit(onSubmit)}>
  ...
</form>
```

---

## Gotchas & Tips

- **`register()` returns `ref` and event handlers** — spread it onto the input. Don't pass `value` (RHF is uncontrolled).
- **Numeric inputs**: use `register("age", { valueAsNumber: true })` or `z.coerce.number()` to convert from string.
- **Default values must be set up-front.** Switching from `undefined` → value re-mounts the input.
- **`mode: "onBlur"` is friendlier than `"onChange"`** — fewer red squigglies while typing.
- **Use `zodResolver` over Yup** for new code: smaller, faster, native TS.
- **Don't validate purely client-side.** Validate again on the server. Zod schemas can be shared between client and server (Next.js/Remix make this easy).
- **`handleSubmit` calls `preventDefault` for you** — no need to do it yourself.
- **Watch perf**: don't call `watch()` for fields that re-render the whole form on every keystroke; use `useWatch` for scoped subscriptions.
- **Accessibility**: pair `<label>` with the input via `htmlFor`/`id` (RHF doesn't help here — that's your job).

---

## See Also

- [[08 - Forms and Inputs]]
- [[19 - TypeScript with React]]
- [[05 - Server Actions]]
- [[11 - Forms at Scale]]
