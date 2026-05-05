---
tags: [react, beginner, jsx]
aliases: [Lists, Keys, .map in JSX]
level: Beginner
---

# Lists and Keys

> **One-liner**: Render a list with `.map()` and give each rendered element a stable, unique `key` so React can match items across renders.

---

## Quick Reference

| Item | Syntax |
|------|--------|
| Render list | `{items.map(i => <li key={i.id}>{i.name}</li>)}` |
| Required `key` | Direct child of the array — not the wrapper |
| Good key | A stable, unique ID from your data (`item.id`) |
| Bad key | Array index when items reorder/insert/delete |
| Worst key | `Math.random()` — re-mounts every render |
| Fragment with key | `<Fragment key={id}>...</Fragment>` (shorthand `<>...</>` can't take key) |

---

## Core Concept

To render a list, return an **array of JSX elements** — usually by calling `.map()` on your data. React renders each element in order.

The **`key`** prop tells React: "this element corresponds to *that* item across renders." When the list changes (items added, removed, reordered), React uses keys to match elements to their previous instance — preserving DOM nodes, internal state, and animations correctly.

A good key is **stable, unique within siblings, and tied to the item's identity**. Database IDs are perfect. The array index is the worst common choice — when an item is inserted at position 0, every key shifts and React thinks every element is "different," which trashes input focus, animations, and child state.

---

## Syntax & API

### Basic list

```tsx
type User = { id: string; name: string };

function UserList({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### Mapping to a component

```tsx
function UserCard({ user }: { user: User }) {
  return <article className="card">{user.name}</article>;
}

function UserGrid({ users }: { users: User[] }) {
  return (
    <div className="grid">
      {users.map(user => <UserCard key={user.id} user={user} />)}
      {/* `key` lives on the JSX element, not inside the component */}
    </div>
  );
}
```

### Multiple elements per item — use a Fragment with key

```tsx
import { Fragment } from "react";

function Glossary({ entries }: { entries: { term: string; def: string }[] }) {
  return (
    <dl>
      {entries.map(e => (
        <Fragment key={e.term}>
          <dt>{e.term}</dt>
          <dd>{e.def}</dd>
        </Fragment>
      ))}
    </dl>
  );
}
```

> The shorthand `<>...</>` does **not** accept props, so when you need a `key` you must use the long form `<Fragment key={...}>`.

### Filter then map

```tsx
const visible = todos.filter(t => !t.done);
return <ul>{visible.map(t => <li key={t.id}>{t.title}</li>)}</ul>;
```

---

## Common Patterns

```tsx
// Pattern: empty state
{items.length === 0
  ? <Empty />
  : items.map(i => <Row key={i.id} item={i} />)}

// Pattern: index-as-key — only safe for static lists that never reorder
{["A", "B", "C"].map((letter, i) => <Tab key={i} label={letter} />)}

// Pattern: composite key when no single field is unique
{rows.map(r => <Cell key={`${r.row}-${r.col}`} cell={r} />)}

// Pattern: re-mount on key change to reset internal state
<EditForm key={selectedUser.id} user={selectedUser} />
// Switching `selectedUser` unmounts the old form (clearing draft state)
```

---

## Gotchas & Tips

- **The `key` warning in the console is real.** It indicates React can't reliably reconcile your list. Fix it.
- **Index-as-key bug**: `key={i}` on a reorderable list causes ghosts of state — the input that was at position 2 now shows the value of the new item at position 2. Symptom: typing into one row appears in another.
- **`Math.random()` as key destroys performance and state.** Every render gets a new key → every item re-mounts.
- **Keys are local to siblings.** Two unrelated lists can both use `key="1"`. Keys don't need to be globally unique.
- **`key` is not passed to the child as a prop** — it's React-only. If the child needs the id, pass it explicitly: `<X key={id} id={id} />`.
- **Don't put `key` on the array's wrapper element.** Put it on each child returned from `.map()`.
- **Stable IDs from data are best.** If your data has no ID, generate one once and store it (don't generate during render).

---

## See Also

- [[02 - JSX Basics]]
- [[06 - Conditional Rendering]]
- [[06 - Performance Optimization]]
