---
tags: [react, beginner, jsx]
aliases: [Events, onClick, onChange]
level: Beginner
---

# Event Handling

> **One-liner**: React wraps native DOM events in **synthetic events** with a consistent cross-browser API; you wire handlers with camelCase props like `onClick={fn}`.

---

## Quick Reference

| Item | Syntax |
|------|--------|
| Click | `<button onClick={fn}>` |
| Change (input) | `<input onChange={e => ...}>` |
| Submit (form) | `<form onSubmit={e => { e.preventDefault(); ... }}>` |
| Key | `onKeyDown` / `onKeyUp` |
| Mouse | `onMouseEnter`, `onMouseLeave`, `onMouseMove` |
| Focus | `onFocus`, `onBlur` |
| Pass argument | `onClick={() => doThing(id)}` |
| Stop propagation | `e.stopPropagation()` |
| Prevent default | `e.preventDefault()` |
| Event type (TS) | `React.MouseEvent<HTMLButtonElement>` |

---

## Core Concept

In React, you attach event listeners by passing a function to a **camelCase event prop** on a JSX element: `<button onClick={handleClick}>`. React doesn't add a real listener to that DOM node — it uses **event delegation** at the root of the React tree, then dispatches a **synthetic event** to your handler.

The synthetic event has the same API as the native event (`e.target`, `e.preventDefault()`, `e.stopPropagation()`) but with cross-browser quirks ironed out. For 99% of cases you can pretend it's a native event.

A handler is just a function. You can define it inline (`onClick={() => ...}`) or extract it for reuse (`onClick={handleSubmit}`). Pass arguments by wrapping in an arrow: `onClick={() => deleteItem(id)}`. Don't write `onClick={deleteItem(id)}` — that *calls* the function during render, then passes the return value as the handler.

---

## Syntax & API

### Click handler

```tsx
function Likes() {
  const [n, setN] = useState(0);

  // Defined separately
  const handleClick = () => setN(n + 1);

  return <button onClick={handleClick}>{n} likes</button>;
}
```

### Inline handler with argument

```tsx
function TodoList({ todos, onDelete }: { todos: Todo[]; onDelete: (id: string) => void }) {
  return (
    <ul>
      {todos.map(t => (
        <li key={t.id}>
          {t.title}
          <button onClick={() => onDelete(t.id)}>delete</button>
        </li>
      ))}
    </ul>
  );
}
```

### Form submit + preventDefault

```tsx
function LoginForm() {
  const [email, setEmail] = useState("");

  const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => {
    e.preventDefault();        // stop the browser from navigating
    console.log("submit:", email);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <button type="submit">Login</button>
    </form>
  );
}
```

### Reading input values (typed)

```tsx
function Search() {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    console.log(e.target.value);
  };

  return <input onChange={handleChange} />;
}
```

### Key events

```tsx
const handleKey = (e: React.KeyboardEvent<HTMLInputElement>) => {
  if (e.key === "Enter") submit();
  if (e.key === "Escape") cancel();
};

<input onKeyDown={handleKey} />;
```

---

## Common Patterns

```tsx
// Pattern: stop event from reaching parents
<div onClick={() => console.log("outer")}>
  <button onClick={e => { e.stopPropagation(); console.log("inner only"); }}>
    inner
  </button>
</div>

// Pattern: one handler, many buttons (closure captures id)
{items.map(item => (
  <button key={item.id} onClick={() => onSelect(item.id)}>
    {item.name}
  </button>
))}

// Pattern: data attribute → no closure per item
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  const id = e.currentTarget.dataset.id;
  onSelect(id);
};

{items.map(item => (
  <button key={item.id} data-id={item.id} onClick={handleClick}>
    {item.name}
  </button>
))}
```

---

## Gotchas & Tips

- **Pass the function, don't call it.** `onClick={fn}` ✅, `onClick={fn()}` ❌ (calls it during render and uses the return value as the handler).
- **`onChange` fires on every keystroke** in React (unlike native HTML, where it fires on blur). React's `onChange` is effectively the native `input` event.
- **Always `preventDefault()` on `<form onSubmit>`** unless you genuinely want a full-page navigation.
- **`e.target` vs `e.currentTarget`**: `target` is what was actually clicked (could be a child); `currentTarget` is the element the listener is on. Prefer `currentTarget` for typed access in TS.
- **Synthetic events are pooled in React <17 only.** In React 17+ they're not pooled, so it's safe to access them async (e.g., inside a `setTimeout`).
- **Handlers re-create on every render.** That's usually fine. Only wrap in `useCallback` when passing to a memoized child (see [[03 - useMemo and useCallback]]).
- **Don't add `onClick` to non-interactive elements** like `<div>`. It breaks keyboard and screen reader users — use `<button>` or add proper ARIA + key handling. See [[20 - Accessibility]].

---

## See Also

- [[04 - State and useState]]
- [[08 - Forms and Inputs]]
- [[19 - TypeScript with React]]
