---
tags: [react, beginner, jsx]
aliases: [JSX]
level: Beginner
---

# JSX Basics

> **One-liner**: JSX is HTML-shaped syntax that compiles to plain JavaScript function calls — it's how React components describe their UI.

---

## Quick Reference

| Item | Syntax |
|------|--------|
| Element | `<h1>Hi</h1>` |
| Self-closing | `<img src="x.png" />` (must close) |
| Expression | `{value}` |
| Attribute | `<a href={url}>` |
| Class attribute | `className` (not `class`) |
| For attribute | `htmlFor` (not `for`) |
| Inline style | `style={{ color: "red" }}` (object, not string) |
| Comment | `{/* like this */}` |
| Fragment | `<>...</>` or `<Fragment>...</Fragment>` |
| Conditional | `{cond && <X />}` or `{cond ? <A /> : <B />}` |
| List | `{items.map(x => <li key={x.id}>{x.name}</li>)}` |

---

## Core Concept

**JSX is not HTML and not a string template.** It's syntactic sugar over `React.createElement(tag, props, ...children)`. Babel/SWC compiles every JSX element into one of those calls before the browser ever sees it.

```tsx
const el = <h1 className="title">Hi</h1>;
// compiles to:
const el = React.createElement("h1", { className: "title" }, "Hi");
```

Because it's JavaScript under the hood, you can put **any expression** between curly braces (`{...}`): variables, function calls, ternaries, `.map()` calls. You **cannot** put statements (`if`, `for`, `let`).

JSX has small but important differences from HTML: attribute names use camelCase (`onClick`, `tabIndex`), `class` becomes `className` (because `class` is a reserved word in JS), `for` becomes `htmlFor`, and self-closing tags are mandatory.

---

## Syntax & API

### Expressions inside JSX

```tsx
function Greeting({ name }: { name: string }) {
  const upper = name.toUpperCase();

  return (
    <div>
      <h1>Hello, {upper}!</h1>
      <p>You have {3 + 4} new messages.</p>
      <p>Today is {new Date().toLocaleDateString()}</p>
    </div>
  );
}
```

### Attributes

```tsx
const url = "/about";
const isActive = true;

<a
  href={url}
  className={isActive ? "link active" : "link"}
  data-id="42"
  aria-label="About page"
>
  About
</a>;
```

### Inline styles (object form)

```tsx
const style = { color: "red", fontSize: 16 }; // camelCase keys

<div style={style}>red</div>;
<div style={{ marginTop: 8, padding: "1rem" }}>literal</div>;
```

### Fragments — group without a wrapper element

```tsx
function Row() {
  return (
    <>
      <td>Name</td>
      <td>Email</td>
    </>
  );
}
```

### Comments

```tsx
<div>
  {/* This is a JSX comment */}
  <p>visible</p>
</div>;
```

---

## Common Patterns

```tsx
// Conditional rendering
{isLoggedIn && <LogoutButton />}
{user ? <Profile user={user} /> : <SignIn />}

// Lists
{users.map(u => <UserCard key={u.id} user={u} />)}

// Inline derived value
<p>Total: ${(price * qty).toFixed(2)}</p>

// Spread props
<input {...inputProps} />
```

---

## Gotchas & Tips

- **Components must return one root element** — use a Fragment (`<>...</>`) if you don't want a wrapper div.
- **`className`, not `class`.** `htmlFor`, not `for`. Editor warnings will catch this.
- **Inline styles take an object**, not a string: `style={{color: "red"}}` (note the **double braces** — outer `{}` for "JS expression", inner `{}` for the object literal).
- **Booleans, `null`, `undefined` render as nothing.** That's why `{cond && <X />}` works — `false` produces no output. But careful: `{0 && <X />}` renders `0` because `0` is falsy but not `null`/`false`.
- **Numbers in `style` are auto-`px`** for length-like properties (`fontSize: 16` → `16px`). Use strings for other units (`"1rem"`, `"50%"`).
- **You can't use `if`/`for` directly inside JSX**, only expressions. Move logic above the `return`, or use a ternary / `&&` / `.map()`.
- **JSX is just JavaScript.** You can assign elements to variables, return them from functions, store them in arrays.

---

## See Also

- [[03 - Components and Props]]
- [[06 - Conditional Rendering]]
- [[07 - Lists and Keys]]
