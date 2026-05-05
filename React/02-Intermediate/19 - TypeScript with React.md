---
tags: [react, intermediate, typescript]
aliases: [TS with React, React TypeScript]
level: Intermediate
---

# TypeScript with React

> **One-liner**: TypeScript catches the most common React mistakes (wrong props, wrong event types, missing fields) at compile time — and once you know a handful of patterns, the friction is minimal.

---

## Quick Reference

| Pattern | Type |
|---------|------|
| Props | `type Props = { name: string }` |
| Children | `children: React.ReactNode` |
| Function component | `(props: Props) => JSX.Element` (return type inferred) |
| Click handler | `(e: React.MouseEvent<HTMLButtonElement>) => void` |
| Change handler | `(e: React.ChangeEvent<HTMLInputElement>) => void` |
| Form submit | `(e: React.FormEvent<HTMLFormElement>) => void` |
| Forwarded ref (≤R18) | `forwardRef<RefType, Props>(...)` |
| State | `useState<User \| null>(null)` |
| Reducer action | discriminated union |
| Pass-through props | `React.ComponentPropsWithoutRef<"button">` |
| `as` prop / polymorphic | generic `E extends React.ElementType` |
| Context value | `createContext<T \| null>(null)` + assertion hook |

---

## Core Concept

TypeScript + React works because React's types are mature and the patterns are well-known. The two areas that trip people up are:
1. **Component props vs DOM props** — when wrapping a `<button>`, you want to accept all native button props plus your own. `React.ComponentPropsWithoutRef<"button">` is the answer.
2. **Generic components** — typing `Select<T>` where item shape varies. Slightly tricky, but the pattern is small and reusable.

Don't over-type. `React.FC` is no longer recommended (implicit `children`, awkward generics) — declare props as a `type` and write the function normally. Let TS infer return types; explicit `JSX.Element` annotation is fine but rarely needed.

When in doubt, hover in your editor and copy the inferred type. The TS tooling for React (in VS Code or any LSP-aware editor) is best-in-class.

---

## Syntax & API

### Function components

```tsx
type GreetingProps = {
  name: string;
  onGreet?: (name: string) => void;
};

function Greeting({ name, onGreet }: GreetingProps) {
  return <button onClick={() => onGreet?.(name)}>Hi, {name}</button>;
}
```

### Children

```tsx
type CardProps = {
  title: string;
  children: React.ReactNode;            // accepts strings, JSX, arrays, null
};
```

### Event handlers

```tsx
const onClick = (e: React.MouseEvent<HTMLButtonElement>) => {
  e.preventDefault();
};

const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  console.log(e.target.value);
};

const onSubmit = (e: React.FormEvent<HTMLFormElement>) => {
  e.preventDefault();
};
```

### `useState` with non-trivial type

```tsx
const [user, setUser] = useState<User | null>(null);
const [items, setItems] = useState<Item[]>([]);
```

### `useReducer` with discriminated union

```tsx
type State = { count: number };
type Action =
  | { type: "inc" }
  | { type: "set"; value: number };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "inc": return { count: state.count + 1 };
    case "set": return { count: action.value };
  }
  // exhaustive switch — TS narrows
}
```

### Wrapping a native element — pass through props

```tsx
type ButtonProps = React.ComponentPropsWithoutRef<"button"> & {
  variant?: "primary" | "secondary";
};

function Button({ variant = "primary", className, ...rest }: ButtonProps) {
  return (
    <button
      className={`btn btn-${variant} ${className ?? ""}`}
      {...rest}
    />
  );
}

// Usage — full button props inferred
<Button onClick={...} disabled type="submit">Save</Button>
```

### `useRef` typing

```tsx
const inputRef = useRef<HTMLInputElement>(null);
inputRef.current?.focus();

// Mutable value (no DOM)
const idRef = useRef<number>(0);   // initialized inline
```

### Generic component (e.g., a `Select<T>`)

```tsx
type SelectProps<T> = {
  items: T[];
  value: T;
  onChange: (v: T) => void;
  getLabel: (item: T) => string;
  getKey:   (item: T) => string;
};

function Select<T>({ items, value, onChange, getLabel, getKey }: SelectProps<T>) {
  return (
    <select
      value={getKey(value)}
      onChange={e => {
        const next = items.find(i => getKey(i) === e.target.value);
        if (next) onChange(next);
      }}
    >
      {items.map(i => (
        <option key={getKey(i)} value={getKey(i)}>
          {getLabel(i)}
        </option>
      ))}
    </select>
  );
}

// Usage with type inference
<Select
  items={users}
  value={selected}
  onChange={setSelected}
  getKey={u => u.id}
  getLabel={u => u.name}
/>
```

### Polymorphic "as" prop

```tsx
type AsProp<E extends React.ElementType> = { as?: E };
type Props<E extends React.ElementType> =
  AsProp<E> &
  Omit<React.ComponentPropsWithoutRef<E>, keyof AsProp<E>>;

function Box<E extends React.ElementType = "div">({ as, ...rest }: Props<E>) {
  const Tag = as || "div";
  return <Tag {...rest} />;
}

<Box>div</Box>
<Box as="a" href="/x">link</Box>
<Box as="button" type="submit">btn</Box>
```

### Context — non-null hook pattern

```tsx
type AuthValue = { user: User | null; logout: () => void };
const AuthCtx = createContext<AuthValue | null>(null);

export function useAuth(): AuthValue {
  const v = useContext(AuthCtx);
  if (!v) throw new Error("useAuth must be inside AuthProvider");
  return v;
}
```

---

## Common Patterns

```tsx
// Pattern: prop type from another component (great for wrappers)
type ModalProps = React.ComponentProps<typeof Dialog> & { onConfirm: () => void };

// Pattern: union prop combos that are valid
type ButtonProps =
  | { variant: "icon"; icon: React.ReactNode; label?: never }
  | { variant: "text"; label: string; icon?: never };

// Pattern: typed event handlers as a single type alias
type ClickHandler = React.MouseEventHandler<HTMLButtonElement>;
```

---

## Gotchas & Tips

- **Don't use `React.FC`.** It silently allows `children`, awkwardly types generics, and is no longer in the React docs. Use `({ ... }: Props) => ...`.
- **`React.ReactNode` is the right children type** in 95% of cases. Use `React.ReactElement` only when you specifically need a single element.
- **`ComponentProps<typeof X>` is gold** for prop-type passthrough.
- **Discriminated unions + `switch` give exhaustiveness checking.** Add a `default: const _: never = action;` line if you want a compiler error on missing cases.
- **`as const` on inferred objects** preserves narrow literal types (useful for action types).
- **`Pick`/`Omit`** are essential — you'll combine and slice prop types constantly.
- **`React.CSSProperties` for inline styles**, `React.AriaAttributes` for ARIA props.
- **For events, inspect what's being targeted** — `React.MouseEvent<HTMLButtonElement>` vs `<HTMLDivElement>`. Wrong target = wrong typed `e.currentTarget`.
- **Generic components need `<T,>` in `.tsx`** (the trailing comma) to disambiguate from JSX. Or use `function X<T>(...)` form which doesn't have this problem.

---

## See Also

- [[03 - Components and Props]]
- [[02 - useRef and forwardRef]]
- [[04 - useReducer]]
- [[05 - useContext]]
