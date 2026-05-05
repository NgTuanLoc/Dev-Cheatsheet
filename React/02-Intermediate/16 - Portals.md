---
tags: [react, intermediate]
aliases: [createPortal, Portal]
level: Intermediate
---

# Portals

> **One-liner**: A **portal** renders children into a DOM node *outside* the parent component's subtree — perfect for modals, tooltips, and toasts that need to escape `overflow: hidden` or `z-index` traps.

---

## Quick Reference

| Item | Syntax |
|------|--------|
| Create portal | `createPortal(children, domNode)` |
| Import | `import { createPortal } from "react-dom"` |
| Target node | Usually `document.body` or a dedicated `#portal-root` |
| React tree position | Stays where the portal **component** is in JSX |
| DOM position | Wherever `domNode` is |
| Event bubbling | Follows React tree, NOT DOM tree |
| Use cases | Modals, drawers, tooltips, toasts, context menus |

---

## Core Concept

React normally renders a child as a sibling/descendant of its parent in the DOM, mirroring the JSX tree. **Portals break that 1:1 mapping**: the child renders inside an arbitrary DOM node, but logically still belongs to the React tree.

Why care? Two reasons:
1. **Visual escape.** A modal nested inside a `position: relative; overflow: hidden` parent gets clipped. Render it in `body` instead — it floats above everything.
2. **Stacking context.** `z-index` is scoped to its stacking context. A tooltip needs to appear above all siblings; portaling to `body` flattens stacking.

The crucial subtle bit: **events bubble through the React tree, not the DOM tree**. A click in a portaled modal still bubbles up through the parent component that rendered it. This is usually what you want (your `onClick` handlers in the parent fire), and it's what `react-error-boundary`, focus traps, and Context all rely on.

---

## Syntax & API

### Minimal modal portal

```tsx
import { createPortal } from "react-dom";

function Modal({ children, onClose }: { children: React.ReactNode; onClose: () => void }) {
  return createPortal(
    <div className="modal-backdrop" onClick={onClose}>
      <div className="modal" onClick={e => e.stopPropagation()}>
        {children}
      </div>
    </div>,
    document.body,
  );
}

// Usage — JSX position is anywhere; DOM position is body
function App() {
  const [open, setOpen] = useState(false);
  return (
    <>
      <button onClick={() => setOpen(true)}>Open</button>
      {open && (
        <Modal onClose={() => setOpen(false)}>
          <p>Hello from a portal</p>
        </Modal>
      )}
    </>
  );
}
```

### Dedicated `#portal-root` (cleaner than body)

```html
<!-- index.html -->
<body>
  <div id="root"></div>
  <div id="portal-root"></div>
</body>
```

```tsx
function PortalHost({ children }: { children: React.ReactNode }) {
  const target = document.getElementById("portal-root");
  if (!target) return null;
  return createPortal(children, target);
}

<PortalHost><Toast text="Saved" /></PortalHost>
```

### Tooltip portal — positioned absolutely

```tsx
function Tooltip({ anchor, text }: { anchor: DOMRect; text: string }) {
  return createPortal(
    <div
      className="tooltip"
      style={{
        position: "absolute",
        top: anchor.bottom + 4,
        left: anchor.left,
      }}
    >
      {text}
    </div>,
    document.body,
  );
}
```

### Lazy-create the target on mount

```tsx
function usePortalTarget() {
  const ref = useRef<HTMLDivElement | null>(null);

  if (!ref.current && typeof document !== "undefined") {
    ref.current = document.createElement("div");
  }

  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    document.body.appendChild(el);
    return () => { document.body.removeChild(el); };
  }, []);

  return ref.current;
}
```

---

## Common Patterns

```tsx
// Pattern: Context still works through portals (React tree, not DOM tree)
<ThemeCtx.Provider value="dark">
  <App>
    <Modal>             {/* portaled — but still inherits theme */}
      <Themed />
    </Modal>
  </App>
</ThemeCtx.Provider>

// Pattern: focus trap on modal open
useEffect(() => {
  const previouslyFocused = document.activeElement as HTMLElement | null;
  modalRef.current?.focus();
  return () => previouslyFocused?.focus();
}, []);
```

---

## Gotchas & Tips

- **Events bubble through the React tree, not the DOM.** A click inside a portaled modal still triggers `onClick` on the modal's React parent — useful, but surprising the first time.
- **Click-outside-to-close** must check the modal's React parent or use a backdrop click handler — `document.addEventListener("click")` will fire even for clicks inside the portaled content.
- **Don't render to a node that doesn't exist yet.** Check `if (!target) return null` for SSR safety.
- **Server rendering**: `createPortal` doesn't work on the server (no `document`). Frameworks usually defer modal rendering to client-only.
- **Accessibility**: a modal needs `role="dialog"`, `aria-modal="true"`, focus management, escape-to-close, and a focus trap. Use a library (Radix UI Dialog, React Aria) instead of rolling your own.
- **Stacking issues** are 90% why people use portals. The other 10% is overflow clipping. If you don't have either problem, you don't need a portal.
- **Cleanup append-to-body nodes** in `useEffect` cleanup — orphaned divs accumulate otherwise.

---

## See Also

- [[17 - Refs and DOM]]
- [[20 - Accessibility]]
- [[14 - Design Systems]]
