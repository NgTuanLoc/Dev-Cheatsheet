---
tags: [react, beginner, forms]
aliases: [Controlled Inputs, Forms]
level: Beginner
---

# Forms and Inputs

> **One-liner**: A **controlled input** stores its value in React state and reads it back via `value=` + `onChange=`; an **uncontrolled input** lets the DOM hold the value and you read it on submit via a `ref` or `FormData`.

---

## Quick Reference

| Pattern | When to use |
|---------|-------------|
| Controlled | You need to validate, transform, or conditionally render based on the value |
| Uncontrolled (`ref` / `FormData`) | Simple submit-only forms, max performance, or integrating non-React DOM libs |
| `<input value=... onChange=...>` | Text, number, search, email, password |
| `<input type="checkbox" checked=... onChange=...>` | Booleans |
| `<input type="radio" checked=... onChange=...>` | One-of-N |
| `<select value=... onChange=...>` | Dropdowns |
| `<textarea value=... onChange=...>` | Multi-line (controlled the same way) |
| `<form onSubmit=...>` | Wraps everything; always `e.preventDefault()` |

---

## Core Concept

In a **controlled input**, React owns the value:

```
state ─→ value=  ─→ <input>
         <input> ─→ onChange ─→ setState
```

Every keystroke fires `onChange`, you `setState`, and the input re-renders with the new `value`. The state is the **single source of truth**.

In an **uncontrolled input**, the DOM owns the value (like vanilla HTML). You either read it on submit via `e.currentTarget.elements.foo.value`, via a `ref`, or via `new FormData(e.currentTarget)`.

For most apps, **controlled inputs are the default** — they make validation, formatting, and conditional UI trivial. Reach for uncontrolled when forms get huge (perf) or when interoperating with non-React widgets. For complex forms, prefer **React Hook Form** (see [[09 - Forms Advanced]]) which uses uncontrolled internally for speed.

---

## Syntax & API

### Controlled text input

```tsx
function NameForm() {
  const [name, setName] = useState("");

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    alert(`Hello, ${name}`);
  };

  return (
    <form onSubmit={handleSubmit}>
      <label>
        Name:
        <input value={name} onChange={e => setName(e.target.value)} />
      </label>
      <button type="submit">Greet</button>
    </form>
  );
}
```

### Multiple fields — one state object

```tsx
type Form = { email: string; password: string };

function LoginForm() {
  const [form, setForm] = useState<Form>({ email: "", password: "" });

  const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: value }));
  };

  return (
    <form>
      <input name="email"    value={form.email}    onChange={onChange} />
      <input name="password" value={form.password} onChange={onChange} type="password" />
    </form>
  );
}
```

### Checkbox & radio

```tsx
const [agreed, setAgreed]   = useState(false);
const [color,  setColor]    = useState<"red" | "blue">("red");

<input
  type="checkbox"
  checked={agreed}
  onChange={e => setAgreed(e.target.checked)}
/>

<input
  type="radio"
  name="color"
  value="red"
  checked={color === "red"}
  onChange={() => setColor("red")}
/>
```

### Select

```tsx
const [size, setSize] = useState("md");

<select value={size} onChange={e => setSize(e.target.value)}>
  <option value="sm">Small</option>
  <option value="md">Medium</option>
  <option value="lg">Large</option>
</select>
```

### Uncontrolled with FormData (lightweight pattern)

```tsx
function ContactForm() {
  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();
    const data = new FormData(e.currentTarget);
    const obj = Object.fromEntries(data.entries());
    console.log(obj); // { name: "...", email: "..." }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" defaultValue="" />
      <input name="email" type="email" defaultValue="" />
      <button type="submit">Send</button>
    </form>
  );
}
```

---

## Common Patterns

```tsx
// Pattern: live transformation (uppercase)
<input
  value={code}
  onChange={e => setCode(e.target.value.toUpperCase())}
/>

// Pattern: numeric input
<input
  type="number"
  value={qty}
  onChange={e => setQty(e.target.valueAsNumber)}
/>

// Pattern: disable submit until valid
<button type="submit" disabled={!email || !password}>
  Submit
</button>
```

---

## Gotchas & Tips

- **Controlled `value` cannot be `undefined`**. React then thinks the input switched from uncontrolled → controlled and warns. Use `value={x ?? ""}` for nullable strings.
- **`defaultValue` is for uncontrolled only.** Mixing `value` and `defaultValue` on the same input is a bug.
- **`onChange` fires every keystroke** — that's fine, but if you do expensive work in the handler, debounce it before `setState` (e.g., for search-as-you-type).
- **`<input type="number">` returns a string** for `e.target.value`. Use `valueAsNumber` (which can be `NaN` if empty).
- **Checkbox uses `checked` + `e.target.checked`**, not `value`.
- **Don't `setState` inside `value={...}`** during render. The handler is the only place state mutates.
- **For complex forms (10+ fields, validation, async)**, switch to React Hook Form + Zod — see [[09 - Forms Advanced]].
- **In React 19, `<form action={fn}>` accepts a function** (Server/Client Action) — see [[05 - Server Actions]].

---

## See Also

- [[05 - Event Handling]]
- [[09 - Forms Advanced]]
- [[02 - useRef and forwardRef]]
