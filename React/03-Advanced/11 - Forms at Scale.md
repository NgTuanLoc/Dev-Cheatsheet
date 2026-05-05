---
tags: [react, advanced, forms]
aliases: [Forms at Scale, Multi-step Forms, Wizard]
level: Advanced
---

# Forms at Scale

> **One-liner**: Real-world forms have multiple steps, server-driven validation, optimistic UI, autosave, accessibility, and progressive enhancement — the 2025 stack is **React Hook Form + Zod (shared client/server) + Server Actions + `useOptimistic`**.

---

## Quick Reference

| Concern | Approach |
|---------|----------|
| Schema | Zod, shared between client & server |
| State | React Hook Form (`useForm`) — uncontrolled = fast |
| Async submit | Server Action via `<form action={fn}>` or RTK/TanStack mutation |
| Server errors | `setError(field, { message })` after action returns |
| Pending UI | `useFormStatus()` (inside form) or RHF `formState.isSubmitting` |
| Optimistic UI | `useOptimistic` (R19) or TanStack Query `onMutate` |
| Multi-step wizard | Single state object across steps; URL or store the step number |
| Autosave | Debounce + `useFetcher` (Remix) or background mutation |
| File upload | `<input type="file">` + `FormData` (works with actions) |
| A11y | Labels, `aria-invalid`, `aria-describedby`, focus the first error |
| No-JS fallback | `<form action>` with a server action — works without hydration |

---

## Core Concept

Once a form has more than 5 fields, validation, or steps, you graduate from `useState` per field to a real form library. The 2025 stack is converging on three pieces:

1. **React Hook Form** for client form state (uncontrolled, minimal re-renders).
2. **Zod** for schema validation — and *the same schema* runs on the server in a Server Action, so you only define rules once.
3. **Server Actions + `useOptimistic`** for submit-and-see-instant-update flows.

Multi-step (wizard) forms add: shared state across steps, URL-driven step progression (so refresh keeps state), and "save partial progress" patterns. Treat the full form as one Zod schema; per-step UIs validate just the current slice via `schema.pick(...)`.

---

## Syntax & API

### Shared schema, client + server

```ts
// schemas/signup.ts — used in both browser and server action
import { z } from "zod";

export const signupSchema = z.object({
  email:     z.string().email(),
  password:  z.string().min(8),
  age:       z.coerce.number().int().min(13),
});
export type SignupInput = z.infer<typeof signupSchema>;
```

### Server action with shared schema (Next.js App Router)

```ts
// app/actions/signup.ts
"use server";

import { signupSchema } from "@/schemas/signup";
import { redirect } from "next/navigation";

export async function signup(_prev: unknown, formData: FormData) {
  const parsed = signupSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { fieldErrors: parsed.error.flatten().fieldErrors };

  const user = await db.user.create({ data: parsed.data });
  redirect(`/welcome/${user.id}`);
}
```

### Client form binding RHF + zodResolver + Server Action

```tsx
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { signupSchema, type SignupInput } from "@/schemas/signup";
import { signup } from "@/app/actions/signup";
import { useActionState } from "react";

export function SignupForm() {
  const [state, formAction, isPending] = useActionState(signup, { fieldErrors: undefined });
  const {
    register,
    handleSubmit,
    setError,
    formState: { errors, isValid },
  } = useForm<SignupInput>({ resolver: zodResolver(signupSchema), mode: "onBlur" });

  // Surface server errors in RHF
  useEffect(() => {
    if (state.fieldErrors) {
      Object.entries(state.fieldErrors).forEach(([k, msgs]) =>
        setError(k as keyof SignupInput, { message: (msgs as string[]).join(", ") })
      );
    }
  }, [state, setError]);

  return (
    <form action={formAction} onSubmit={handleSubmit(() => {})}>
      <Field name="email"    label="Email"    register={register} error={errors.email} />
      <Field name="password" label="Password" register={register} error={errors.password} type="password" />
      <Field name="age"      label="Age"      register={register} error={errors.age} type="number" />
      <button type="submit" disabled={isPending || !isValid}>
        {isPending ? "Creating…" : "Sign up"}
      </button>
    </form>
  );
}
```

### Multi-step wizard

```tsx
const fullSchema = z.object({
  account:   accountSchema,
  profile:   profileSchema,
  payment:   paymentSchema,
});

type Wizard = z.infer<typeof fullSchema>;

const steps = ["account", "profile", "payment"] as const;
type Step = typeof steps[number];

function Wizard() {
  const [step, setStep] = useState<Step>("account");
  const [data, setData] = useState<Partial<Wizard>>({});

  const onStepSubmit = (slice: Partial<Wizard>) => {
    const next = { ...data, ...slice };
    setData(next);
    const idx = steps.indexOf(step);
    if (idx < steps.length - 1) setStep(steps[idx + 1]);
    else submitFinal(next);
  };

  return (
    <>
      <Stepper current={step} />
      {step === "account" && <AccountStep   defaults={data.account}   onNext={d => onStepSubmit({ account: d })} />}
      {step === "profile" && <ProfileStep   defaults={data.profile}   onNext={d => onStepSubmit({ profile: d })} />}
      {step === "payment" && <PaymentStep   defaults={data.payment}   onNext={d => onStepSubmit({ payment: d })} />}
    </>
  );
}
```

### Autosave (Remix `useFetcher` debounced)

```tsx
function NotesEditor({ noteId, initial }: { noteId: string; initial: string }) {
  const [draft, setDraft] = useState(initial);
  const fetcher = useFetcher();
  const debounced = useDebounced(draft, 500);

  useEffect(() => {
    if (debounced === initial) return;
    const fd = new FormData();
    fd.set("body", debounced);
    fetcher.submit(fd, { method: "post", action: `/notes/${noteId}` });
  }, [debounced, initial, noteId, fetcher]);

  return <textarea value={draft} onChange={e => setDraft(e.target.value)} />;
}
```

### Optimistic submit with `useOptimistic`

```tsx
"use client";
import { useOptimistic, useTransition } from "react";
import { addItem } from "./actions";

function ItemList({ initial }: { initial: Item[] }) {
  const [optimistic, addOptimistic] = useOptimistic(initial, (state, t: Item) => [...state, t]);
  const [, start] = useTransition();

  return (
    <>
      <ul>{optimistic.map(i => <li key={i.id}>{i.title}</li>)}</ul>
      <form action={(fd) => {
        const title = fd.get("title") as string;
        start(async () => {
          addOptimistic({ id: crypto.randomUUID(), title });
          await addItem(fd);
        });
      }}>
        <input name="title" />
      </form>
    </>
  );
}
```

---

## Common Patterns

```tsx
// Pattern: focus first invalid field on submit fail
useEffect(() => {
  const firstError = Object.keys(errors)[0];
  if (firstError) document.querySelector<HTMLInputElement>(`[name="${firstError}"]`)?.focus();
}, [errors]);

// Pattern: per-step validation slice from full schema
const accountSchema = fullSchema.pick({ account: true });
```

---

## Gotchas & Tips

- **Share schemas across boundaries.** Validating only client-side is a security bug; only server-side gives bad UX.
- **Server-side validation is authoritative.** Client validation is a UX optimization.
- **Don't lose draft state on navigation.** Either persist (localStorage) or hold it in URL/store.
- **File uploads** in actions: use `FormData`, set `enctype="multipart/form-data"` (the Form helper does it). Big uploads need streaming; consider direct-to-S3 with presigned URLs.
- **Never disable submit on `isValid` alone**; also block while `isSubmitting`.
- **Wizards: validate cumulative state before final submit** — a step-3 user can edit step 1 and break it.
- **Autosave must be debounced and idempotent** — the server should accept the same partial multiple times.
- **Accessibility**: announce errors with `role="alert"` or in a live region; focus the first invalid field; use `aria-describedby` for hint text.
- **Progressive enhancement** (no-JS form action) is a free win with Server Actions — keep your forms working before hydration.

---

## See Also

- [[09 - Forms Advanced]]
- [[05 - Server Actions]]
- [[20 - Accessibility]]
- [[11 - TanStack Query]]
