---
tags: [react, beginner, styling]
aliases: [CSS in React, Styling]
level: Beginner
---

# Styling

> **One-liner**: React doesn't pick a styling approach for you — the common options are **CSS modules**, **Tailwind CSS**, **CSS-in-JS** (emotion / styled-components), and plain CSS files; in 2025 most new projects use Tailwind or CSS modules.

---

## Quick Reference

| Approach | What it is | Sweet spot |
|----------|-----------|------------|
| **Plain CSS** | Import a `.css` file | Tiny apps, prototypes |
| **CSS modules** | `styles.module.css` — locally scoped class names | Library-friendly, framework-agnostic |
| **Tailwind CSS** | Utility classes in `className` | Fast iteration, design systems |
| **CSS-in-JS** | Styles defined in JS (emotion, styled-components) | Dynamic theming; fading in 2025 |
| **Vanilla-Extract** | Type-safe CSS-in-TS, zero runtime | Modern alternative to CSS-in-JS |
| **Inline `style`** | Object literal | One-off dynamic values only |

---

## Core Concept

React renders elements; **how you style them is a separate concern**. The DOM understands two things: a `class` attribute (which React calls `className`) and inline `style` (an object). Every styling library boils down to setting one of those two.

The big tradeoffs:

- **Plain CSS / global stylesheets** — simple, but class names collide.
- **CSS modules** — same files, but each class name gets a hashed suffix at build time, scoped to the importing component.
- **Tailwind** — never write CSS files. Compose utility classes (`flex`, `gap-4`, `text-blue-500`) directly in markup. Trades JSX verbosity for zero context-switching.
- **CSS-in-JS** — write styles as JS strings/objects. Powerful but pays a runtime cost; declining in popularity since RSC + Tailwind ate its lunch.

For new projects: **Tailwind** if your team likes utility-first; **CSS modules** if you want plain CSS with scope. Inline `style` only for truly dynamic values (computed positions, animated colors).

---

## Syntax & API

### Plain CSS (global)

```css
/* App.css */
.title { color: navy; font-size: 2rem; }
```

```tsx
// App.tsx
import "./App.css";

export default function App() {
  return <h1 className="title">Hello</h1>;
}
```

### CSS modules (scoped)

```css
/* Button.module.css */
.button { padding: 0.5rem 1rem; border-radius: 4px; }
.primary { background: navy; color: white; }
```

```tsx
import styles from "./Button.module.css";

export function Button({ primary }: { primary?: boolean }) {
  return (
    <button className={`${styles.button} ${primary ? styles.primary : ""}`}>
      Click
    </button>
  );
}
```

### Tailwind CSS

```bash
npm install -D tailwindcss @tailwindcss/vite
# Vite: add @tailwindcss/vite plugin to vite.config.ts
```

```css
/* index.css */
@import "tailwindcss";
```

```tsx
function Card({ children }: { children: React.ReactNode }) {
  return (
    <div className="rounded-lg border border-gray-200 bg-white p-4 shadow-sm">
      {children}
    </div>
  );
}
```

### Conditional class names — `clsx` is the standard helper

```tsx
import clsx from "clsx";

<button
  className={clsx(
    "btn",
    primary && "btn-primary",
    disabled && "btn-disabled",
    size === "lg" && "btn-lg",
  )}
/>
```

### Inline style (object — camelCase keys)

```tsx
<div style={{
  width: 200,                // number → "200px"
  marginTop: "1rem",
  backgroundColor: color,    // dynamic
}}>
  ...
</div>
```

---

## Common Patterns

```tsx
// Pattern: variant prop with Tailwind
const variants = {
  primary:   "bg-blue-600 text-white hover:bg-blue-700",
  secondary: "bg-gray-200 text-gray-900 hover:bg-gray-300",
  danger:    "bg-red-600 text-white hover:bg-red-700",
};

function Button({ variant = "primary", className, ...rest }: Props) {
  return (
    <button
      className={clsx("rounded px-4 py-2 font-medium", variants[variant], className)}
      {...rest}
    />
  );
}
```

```tsx
// Pattern: theming with CSS variables (works with any styling)
:root {
  --color-bg: #fff;
  --color-fg: #111;
}
[data-theme="dark"] {
  --color-bg: #111;
  --color-fg: #fff;
}

// In JSX
<html data-theme={theme}>
  ...
</html>
```

---

## Gotchas & Tips

- **`className`, not `class`.** A common copy-from-HTML mistake.
- **Inline style numbers default to px** for length-like properties. Strings keep your unit (`"50%"`, `"2rem"`).
- **Don't recreate large `style` objects every render** if a child is memoized — extract or memoize them.
- **Tailwind class names must appear as full literal strings** for the JIT to detect them. `bg-${color}-500` won't work — use a lookup map.
- **CSS-in-JS adds runtime cost and breaks Server Components** (most CSS-in-JS libs use the React `style` element trick that doesn't survive RSC). Prefer Tailwind, CSS modules, or vanilla-extract for new RSC apps.
- **Global resets only once.** A `* { box-sizing: border-box }` belongs in your root `index.css`, not in components.
- **Use `clsx` (or `cn`)**. Long template literals with conditional classes are unreadable.

---

## See Also

- [[03 - Components and Props]]
- [[14 - Design Systems]]
- [[12 - Animations]]
