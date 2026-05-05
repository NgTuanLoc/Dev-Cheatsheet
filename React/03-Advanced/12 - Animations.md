---
tags: [react, advanced, animation, styling]
aliases: [Framer Motion, motion, Animations]
level: Advanced
---

# Animations

> **One-liner**: Animate in React with **CSS transitions** for trivial cases, **Framer Motion** (now `motion/react`) for layout/exit/gesture animations, and respect `prefers-reduced-motion` for users who get sick from movement.

---

## Quick Reference

| Need | Tool |
|------|------|
| Hover/focus state change | CSS `:hover`, `:focus`, `transition` |
| Mount/unmount fade | `<AnimatePresence>` from Framer Motion |
| Layout transitions (FLIP) | `<motion.div layout>` |
| Spring/inertia physics | `transition={{ type: "spring" }}` |
| Drag, gesture | `<motion.div drag>` |
| Scroll-linked | `useScroll`, `useTransform` |
| Lottie / After Effects | `lottie-react` |
| 3D / WebGL | `@react-three/fiber` (R3F) |
| Reduced motion | `useReducedMotion()` (Framer) or CSS media query |

---

## Core Concept

Most "animation" in React apps is just CSS transitions on opacity/transform — fast, native, no library needed. Reach for an animation library when you need:

- **Mount / unmount transitions** (React doesn't expose unmount lifecycle to CSS easily; `AnimatePresence` solves it).
- **Layout animations** (FLIP technique — animate from old to new layout when DOM changes).
- **Physics** (spring, inertia, drag).
- **Gesture handling** (drag, swipe, pan with momentum).

**Framer Motion** (rebranded **`motion/react`** in 2024) is the de-facto choice. The API is declarative — you set `animate`, `initial`, `exit`, `transition` props on `motion.X` elements, and the library handles the imperative dance.

For complex sequences (hero animations, scroll choreography), GSAP and Theatre.js still win. For 3D scenes, React Three Fiber.

**Always honor `prefers-reduced-motion`.** Users with vestibular disorders should not be motion-sick from your hover effect.

---

## Syntax & API

### CSS-only transition (when it suffices)

```css
.btn {
  transition: background 150ms ease, transform 100ms;
}
.btn:hover { background: navy; transform: translateY(-1px); }
```

### Framer Motion — fade in on mount

```bash
npm install motion
```

```tsx
import { motion } from "motion/react";

function Card() {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3 }}
    >
      Hello
    </motion.div>
  );
}
```

### Exit animation with `AnimatePresence`

```tsx
import { motion, AnimatePresence } from "motion/react";

function Modal({ open, children }: { open: boolean; children: React.ReactNode }) {
  return (
    <AnimatePresence>
      {open && (
        <motion.div
          key="modal"
          initial={{ opacity: 0, scale: 0.96 }}
          animate={{ opacity: 1, scale: 1 }}
          exit={{ opacity: 0, scale: 0.96 }}
          transition={{ duration: 0.18 }}
        >
          {children}
        </motion.div>
      )}
    </AnimatePresence>
  );
}
```

### Layout animations (FLIP — auto-animate layout changes)

```tsx
{items.map(item => (
  <motion.li key={item.id} layout transition={{ type: "spring", stiffness: 500, damping: 30 }}>
    {item.name}
  </motion.li>
))}
// Reordering / inserting / removing items now animates smoothly.
```

### Drag

```tsx
<motion.div
  drag="x"
  dragConstraints={{ left: -100, right: 100 }}
  dragElastic={0.2}
  whileTap={{ scale: 1.05 }}
>
  drag me
</motion.div>
```

### Scroll-linked animation

```tsx
import { useScroll, useTransform, motion } from "motion/react";

function ProgressBar() {
  const { scrollYProgress } = useScroll();
  const width = useTransform(scrollYProgress, [0, 1], ["0%", "100%"]);
  return (
    <motion.div
      style={{ width, height: 4, background: "deeppink", position: "fixed", top: 0 }}
    />
  );
}
```

### Reduced motion

```tsx
import { useReducedMotion } from "motion/react";

function Hero() {
  const reduce = useReducedMotion();
  return (
    <motion.h1
      initial={reduce ? false : { opacity: 0, y: 30 }}
      animate={{ opacity: 1, y: 0 }}
    >
      Title
    </motion.h1>
  );
}
```

```css
/* CSS equivalent */
@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}
```

---

## Common Patterns

```tsx
// Pattern: stagger children
<motion.ul
  initial="hidden"
  animate="show"
  variants={{ show: { transition: { staggerChildren: 0.05 } } }}
>
  {items.map(i => (
    <motion.li
      key={i.id}
      variants={{ hidden: { opacity: 0 }, show: { opacity: 1 } }}
    >
      {i.name}
    </motion.li>
  ))}
</motion.ul>

// Pattern: shared layout animation between two elements (hero → detail)
<motion.div layoutId={`thumb-${id}`} />     {/* in list */}
<motion.div layoutId={`thumb-${id}`} />     {/* on detail page — animates between */}

// Pattern: spring presets
transition={{ type: "spring", stiffness: 260, damping: 20 }}
```

---

## Gotchas & Tips

- **Animate `transform` and `opacity`**, not `width`/`top`/`left`. They're GPU-accelerated; the others trigger layout/paint.
- **`AnimatePresence` requires unique `key`s** on direct children — that's how it tracks what's exiting.
- **Layout animations re-measure on every layout change.** They're cheap, but don't enable on lists of thousands of items.
- **Reduced motion**: respect it. Library helpers (`useReducedMotion`) make this one line.
- **Test on a slow device.** A 90fps spring on your laptop may be 20fps on a low-end Android.
- **`will-change: transform` hints to the browser** — can help, but overuse hurts memory. Default to no.
- **For complex timelines**, GSAP's primitives are unmatched. For React-y declarative motion, Framer is friendlier.
- **Lottie** is great for designer-handed-off animations from After Effects — no React work needed beyond mounting the player.
- **Don't animate too much.** Subtle motion is professional; busy motion is amateurish.

---

## See Also

- [[09 - Styling]]
- [[20 - Accessibility]]
- [[14 - Design Systems]]
