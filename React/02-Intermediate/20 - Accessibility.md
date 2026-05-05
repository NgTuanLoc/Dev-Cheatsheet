---
tags: [react, intermediate, accessibility]
aliases: [a11y, ARIA, Accessibility]
level: Intermediate
---

# Accessibility

> **One-liner**: Accessibility (a11y) means your app works for users on screen readers, keyboard-only, low vision, motor impairment — and the foundation is just **semantic HTML**, with ARIA only filling in gaps the platform doesn't have.

---

## Quick Reference

| Rule | Mean |
|------|------|
| **Semantic HTML first** | `<button>`, `<nav>`, `<main>`, `<label>` carry built-in a11y |
| **Every form input has a label** | `<label htmlFor="x">` + `<input id="x">` |
| **Interactive elements are focusable** | Use `<button>`, not `<div onClick>` |
| **Keyboard works** | Tab, Shift-Tab, Enter, Space, Arrows, Escape |
| **Visible focus** | Don't `outline: none` without an alternative |
| **Color contrast** | ≥ 4.5:1 for body text, 3:1 for large |
| **`alt` on images** | Empty `alt=""` for decorative; descriptive for meaningful |
| **`role`/`aria-*`** | Only when no semantic element fits |
| **Auto-test** | `axe-core`, `eslint-plugin-jsx-a11y` |
| **Manual test** | Tab through it; close your eyes and use VoiceOver/NVDA |

---

## Core Concept

90% of accessibility is **using the right HTML element**. A `<button>` gets focus, fires on Enter/Space, exposes role and label automatically. A `<div onClick>` does none of that, and you have to reinvent everything (`tabIndex`, `role`, `onKeyDown`).

ARIA is a layer on top — it lets you describe widgets that HTML doesn't have (`role="combobox"`, `aria-expanded`). The **first rule of ARIA**: don't use ARIA when a native element exists. The **second rule**: don't change a native element's semantics.

Real accessibility happens at three points:
1. **Markup** — semantic, labeled, ordered.
2. **Keyboard** — every interaction reachable without a mouse.
3. **Visible focus** — keyboard users need to see where they are.

Tools:
- `eslint-plugin-jsx-a11y` catches missing labels, bad roles, click-without-keyboard, etc., at lint time.
- `axe-core` (or `@axe-core/react`) scans rendered output for WCAG violations.
- Browsers' built-in DevTools (Chrome Lighthouse, Firefox Accessibility Inspector) audit pages.

---

## Syntax & API

### Semantic structure

```tsx
function Page() {
  return (
    <>
      <header><Logo /></header>
      <nav aria-label="Primary"><MainNav /></nav>
      <main>
        <h1>Page title</h1>
        <article>...</article>
      </main>
      <footer>...</footer>
    </>
  );
}
```

### Labels — three valid forms

```tsx
// 1. Wrapping <label>
<label>
  Email
  <input type="email" />
</label>

// 2. htmlFor + id
<label htmlFor="email">Email</label>
<input id="email" type="email" />

// 3. aria-label (when no visible label is appropriate, e.g., icon button)
<button aria-label="Close" onClick={onClose}>
  <XIcon aria-hidden="true" />
</button>
```

### Custom interactive — only if you must

```tsx
// Don't do this if a <button> works:
<div
  role="button"
  tabIndex={0}
  onClick={onClick}
  onKeyDown={(e) => {
    if (e.key === "Enter" || e.key === " ") {
      e.preventDefault();
      onClick();
    }
  }}
>
  Click me
</div>
```

### Focus management — on modal open

```tsx
function Modal({ children, onClose }: { children: React.ReactNode; onClose: () => void }) {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const previouslyFocused = document.activeElement as HTMLElement | null;
    ref.current?.focus();
    return () => previouslyFocused?.focus();   // restore on close
  }, []);

  useEffect(() => {
    const onKey = (e: KeyboardEvent) => { if (e.key === "Escape") onClose(); };
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [onClose]);

  return (
    <div role="dialog" aria-modal="true" tabIndex={-1} ref={ref}>
      {children}
    </div>
  );
}
```

### Live region — announce updates

```tsx
<div role="status" aria-live="polite">
  {savedMessage && "Changes saved"}
</div>
```

### `prefers-reduced-motion`

```tsx
const prefersReduced = useMediaQuery("(prefers-reduced-motion: reduce)");

<div className={prefersReduced ? "" : "animate-fade-in"}>...</div>
```

### Skip link (keyboard users)

```tsx
<a href="#main" className="sr-only focus:not-sr-only">Skip to main content</a>
<main id="main" tabIndex={-1}>...</main>
```

---

## Common Patterns

```tsx
// Pattern: icon-only button with name
<button aria-label="Delete item" onClick={onDelete}>
  <TrashIcon aria-hidden="true" />
</button>

// Pattern: ARIA describedby for hint text
<label htmlFor="pwd">Password</label>
<input id="pwd" type="password" aria-describedby="pwd-hint" />
<p id="pwd-hint">At least 8 characters.</p>

// Pattern: form error association
<input aria-invalid={!!error} aria-describedby={error ? "email-err" : undefined} />
{error && <p id="email-err" role="alert">{error}</p>}
```

---

## Gotchas & Tips

- **`<div onClick>` is the most common a11y bug.** It's not focusable, not keyboard-operable, not announced as interactive. Use `<button>`.
- **`outline: none` without replacement is illegal under WCAG.** Provide `:focus-visible` styling.
- **`tabIndex={0}` adds to tab order; `tabIndex={-1}` makes focusable but not in order** (for programmatic focus, e.g., modal containers).
- **Don't use `tabIndex` > 0** — it scrambles natural order.
- **Color alone never conveys meaning.** Pair with text/icon for colorblind users.
- **Headings must be hierarchical.** One `<h1>` per page; don't skip levels (`h2` → `h4`).
- **`alt=""` for decorative images** — screen readers skip them. Missing `alt` makes them announce the file URL.
- **For complex widgets (combobox, listbox, tree, tabs)**, use a battle-tested library — Radix UI, React Aria, Headless UI. Hand-rolling ARIA is a minefield.
- **Test with a real screen reader** at least once — VoiceOver (macOS, Cmd+F5), NVDA (Windows, free), Narrator (Windows, built-in). Reveals issues no auditor catches.
- **`eslint-plugin-jsx-a11y` is mandatory** for any team — catches the easy 50% in your editor.

---

## See Also

- [[05 - Event Handling]]
- [[16 - Portals]]
- [[14 - Design Systems]]
