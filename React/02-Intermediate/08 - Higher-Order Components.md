---
tags: [react, intermediate, architecture]
aliases: [HOC, withX]
level: Intermediate
---

# Higher-Order Components

> **One-liner**: A **HOC** is a function `Component → Component` that wraps another component to inject props or behavior — a pre-hooks pattern that's mostly replaced by **custom hooks**, but still useful for cross-cutting concerns like authentication wrappers, error boundaries, or DOM-manipulating wrappers.

---

## Quick Reference

| Item | Form |
|------|------|
| Signature | `function withX<P>(Wrapped: ComponentType<P>): ComponentType<P>` |
| Naming | `withAuth`, `withRouter`, `withTheme` |
| Display name | Set `Wrapped.displayName` for DevTools clarity |
| Forward refs | Wrap with `forwardRef` so refs reach `Wrapped` |
| Forward props | Always `{...props}` — never drop them |
| When to use | Wrapping (Suspense + ErrorBoundary), permission gates, DOM observers |
| When to use **hooks instead** | Sharing stateful logic that doesn't change rendering |

---

## Core Concept

Before hooks (React <16.8), the only way to share stateful logic between components was either render props or HOCs. A HOC is just a **function that returns a new component**, usually wrapping the original to add props or behavior.

```
Component → withSomething(Component) → EnhancedComponent
```

Today (2025), most things you'd write a HOC for are better written as a **custom hook**. Hooks are simpler, less indirection, and don't pollute the component tree with `withX(withY(withZ(Component)))`.

HOCs still earn their keep when:
- You need to **wrap with rendering** that doesn't depend on the component (auth gate, suspense+error boundary).
- You're integrating with non-React libraries (Redux's old `connect`, MobX's `inject`).
- You need to **conditionally render** the inner component (route guards).

---

## Syntax & API

### Minimal HOC

```tsx
import { ComponentType } from "react";

function withLogging<P extends object>(Wrapped: ComponentType<P>) {
  function Wrapper(props: P) {
    console.log("rendering", Wrapped.displayName ?? Wrapped.name, props);
    return <Wrapped {...props} />;
  }
  Wrapper.displayName = `withLogging(${Wrapped.displayName ?? Wrapped.name})`;
  return Wrapper;
}

const LoggedButton = withLogging(Button);
```

### Auth gate HOC

```tsx
function withAuth<P extends object>(Wrapped: ComponentType<P>) {
  return function Guarded(props: P) {
    const { user, status } = useAuth();
    if (status === "loading") return <Spinner />;
    if (!user) return <LoginPage />;
    return <Wrapped {...props} />;
  };
}

export default withAuth(Dashboard);
```

### HOC that injects a prop

```tsx
type WithUser = { user: User };

function withUser<P extends WithUser>(Wrapped: ComponentType<P>) {
  return function WithUser(props: Omit<P, "user">) {
    const user = useCurrentUser();
    if (!user) return null;
    return <Wrapped {...(props as P)} user={user} />;
  };
}

// Wrapped expects { user: User; ...other }
// Consumer passes only ...other; HOC supplies user.
```

### HOC that forwards refs

```tsx
import { forwardRef } from "react";

function withTooltip<P extends object>(Wrapped: ComponentType<P>) {
  return forwardRef<unknown, P & { tip?: string }>(function Wrapper(props, ref) {
    const { tip, ...rest } = props;
    return (
      <span title={tip}>
        <Wrapped ref={ref as any} {...(rest as P)} />
      </span>
    );
  });
}
```

---

## Common Patterns

```tsx
// Pattern: HOC ↔ hook duality
// HOC version
function withWindowSize<P extends { size: { w: number; h: number } }>(Wrapped: ComponentType<P>) {
  return function (props: Omit<P, "size">) {
    const size = useWindowSize();
    return <Wrapped {...(props as P)} size={size} />;
  };
}

// Hook version (simpler)
function MyComponent() {
  const size = useWindowSize();
  return <div>{size.w} × {size.h}</div>;
}

// → prefer the hook unless you really need the HOC for wrapping logic.
```

```tsx
// Pattern: composing HOCs
const enhance = (C: ComponentType<any>) => withAuth(withLogging(withTheme(C)));
export default enhance(Dashboard);
// Tradeoff: shorter at use site, but DevTools tree is deeper.
```

---

## Gotchas & Tips

- **Don't lose props.** Always spread `{...props}` (or carefully omit only what you intend).
- **Set `displayName`.** Otherwise DevTools shows `Anonymous` everywhere — debugging nightmare.
- **Don't forget `forwardRef`** if the wrapped component might receive a `ref`. Otherwise `ref` is silently dropped.
- **Don't compose more than 2–3 HOCs.** "wrapper hell" makes stack traces unreadable.
- **HOC vs hook**: if your "HOC" is just sharing logic and adding a prop, it's almost certainly clearer as a custom hook.
- **HOCs run during render.** Don't put side effects in the wrapper — use effects inside.
- **Library examples**: `connect()` from old Redux, `inject()` from MobX, `withRouter()` from React Router 5, `withErrorBoundary()` from `react-error-boundary`. All have hook equivalents in their modern versions (`useSelector`, `useObserver`, `useNavigate`, `useErrorHandler`).

---

## See Also

- [[06 - Custom Hooks]]
- [[07 - Component Composition]]
- [[15 - Error Boundaries]]
