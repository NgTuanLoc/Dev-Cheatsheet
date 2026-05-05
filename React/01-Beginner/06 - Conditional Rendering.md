---
tags: [react, beginner, jsx]
aliases: [Conditional Rendering, If in JSX]
level: Beginner
---

# Conditional Rendering

> **One-liner**: JSX has no `if` statement — you express conditions with **expressions**: ternaries (`?:`), logical AND (`&&`), early returns, and lookup objects.

---

## Quick Reference

| Pattern | Use when |
|---------|----------|
| Early `return null` | Whole component should render nothing |
| `cond && <X />` | Render `<X />` only if `cond` is truthy |
| `cond ? <A /> : <B />` | Choose between two branches |
| `cond ? <A /> : null` | Same as `&&` but explicit (clearer in some cases) |
| Lookup object | Many discrete cases (statuses, types) |
| `if` above `return` | Anything more complex than a ternary |

---

## Core Concept

JSX accepts **JavaScript expressions** inside `{}`, but not statements. So you can't write `{ if (x) ... }`. The four common expressions are: ternary, logical AND, function call returning JSX, and a value (variable or null).

The right tool depends on shape:
- One branch only? → `cond && <X />`
- Two branches? → ternary `cond ? <A /> : <B />`
- Many cases? → object lookup, or extract a function with `if`/`switch`
- Whole component should disappear? → `if (!ready) return null` at the top

**`null`, `false`, `undefined`, and `true` render as nothing.** That's why `&&` works: when the left side is falsy, the expression is `false`, which renders nothing.

---

## Syntax & API

### Early return — bail out before rendering

```tsx
function UserPanel({ user }: { user: User | null }) {
  if (!user) return null;        // render nothing
  // ...rest only runs when user exists
  return <h1>Hello, {user.name}</h1>;
}
```

### Logical AND (single branch)

```tsx
function Inbox({ unread }: { unread: number }) {
  return (
    <header>
      Inbox
      {unread > 0 && <span className="badge">{unread}</span>}
    </header>
  );
}
```

### Ternary (two branches)

```tsx
function AuthButton({ user }: { user: User | null }) {
  return user
    ? <button onClick={logout}>Logout</button>
    : <button onClick={login}>Login</button>;
}
```

### Lookup object (many cases — cleaner than long ternaries)

```tsx
type Status = "idle" | "loading" | "success" | "error";

const STATUS_ICON: Record<Status, string> = {
  idle: "⏸",
  loading: "⏳",
  success: "✅",
  error: "❌",
};

function StatusBadge({ status }: { status: Status }) {
  return <span>{STATUS_ICON[status]}</span>;
}
```

### Helper function (when JSX gets ugly)

```tsx
function renderBody(state: State) {
  if (state.loading) return <Spinner />;
  if (state.error)   return <ErrorView error={state.error} />;
  if (!state.data)   return <Empty />;
  return <DataView data={state.data} />;
}

function Page() {
  const state = useFetch();
  return (
    <main>
      <Header />
      {renderBody(state)}
    </main>
  );
}
```

---

## Common Patterns

```tsx
// Pattern: list with empty state
{items.length === 0
  ? <Empty message="No results" />
  : items.map(i => <Row key={i.id} item={i} />)}

// Pattern: feature flag
{flags.newCheckout && <NewCheckout />}

// Pattern: loading + error + data, in one return
return (
  <>
    {loading && <Spinner />}
    {error && <ErrorBanner error={error} />}
    {data && <Table rows={data} />}
  </>
);
```

---

## Gotchas & Tips

- **`&&` with numbers prints the number.** `{count && <X />}` renders `0` when `count === 0` because `0` is falsy but renders as text. Use `{count > 0 && <X />}` or a ternary.
- **`null` is fine, `undefined` is fine, but avoid leaking `false` as text** — it renders as nothing, but reviewers find it confusing. Be explicit with `cond ? <X /> : null`.
- **Don't put expensive work in conditionals during render.** `{computeExpensiveThing() && <X />}` runs on every render. Memoize or move it out.
- **Avoid deeply nested ternaries.** `a ? b : c ? d : e ? f : g` is unreadable. Extract to a helper.
- **Conditional hooks are forbidden.** You can `if (...) return <X />` to render conditionally, but you cannot `if (cond) useState(...)`. Hooks must run in the same order every render. See [[06 - Custom Hooks]].
- **Conditional `key` resets state.** Mounting `<X key={a} />` then `<X key={b} />` unmounts and remounts — useful trick to reset internal state.

---

## See Also

- [[02 - JSX Basics]]
- [[07 - Lists and Keys]]
- [[15 - Error Boundaries]]
