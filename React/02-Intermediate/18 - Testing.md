---
tags: [react, intermediate, testing]
aliases: [Testing, RTL, Vitest]
level: Intermediate
---

# Testing

> **One-liner**: Test React components the way users use them — render, query by **accessible role/text**, simulate events with `user-event`, assert visible behavior — using **Vitest** (or Jest) + **React Testing Library** + **MSW** for API mocking.

---

## Quick Reference

| Tool | Purpose |
|------|---------|
| **Vitest** | Fast Vite-native test runner (Jest-compatible API) |
| **Jest** | Older standard test runner |
| **React Testing Library (RTL)** | Render + query DOM the user-facing way |
| **`@testing-library/user-event`** | Realistic event simulation (typing, clicks, focus) |
| **MSW** (Mock Service Worker) | Intercept network calls without mocking `fetch` |
| **Playwright / Cypress** | E2E in a real browser |
| **`renderHook`** | Test custom hooks in isolation |

| Query priority | Most → least preferred |
|----------------|------------------------|
| `getByRole`     | role + accessible name (best — mirrors a11y) |
| `getByLabelText` | form fields |
| `getByText`     | non-interactive text |
| `getByTestId`   | last resort — not user-visible |

---

## Core Concept

The Testing Library philosophy: **test the way users interact, not internals**. Don't assert on component state, props, or implementation details — assert on what the user *sees* and what *happens* when they interact.

That means:
- **Render** with `render(<Component />)`.
- **Find** elements by their accessible role/label/text — the same things screen readers and users see.
- **Interact** via `user-event` (`await user.type`, `await user.click`).
- **Assert** with `expect(screen.getByText(...))`. Use `findByX` (returns a Promise) for things that appear async.

If a test is hard to write with RTL, the component is probably hard to use. RTL pushes you toward accessible markup as a side effect.

For network calls, **don't mock `fetch` directly**. Use **MSW** to intercept HTTP at the network layer — your component code stays untouched, and the same mocks work in dev/Storybook/tests.

---

## Diagram

```mermaid
flowchart LR
    A[render(JSX)] --> B[query elements<br/>getByRole, getByText]
    B --> C[user.click / user.type]
    C --> D[expect(screen.getBy...).toBeInTheDocument]
```

---

## Syntax & API

### Setup (Vitest + RTL)

```bash
npm install -D vitest @vitest/ui jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    setupFiles: ["./test-setup.ts"],
    globals: true,
  },
});
```

```ts
// test-setup.ts
import "@testing-library/jest-dom/vitest";
```

### A simple test

```tsx
// Counter.test.tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Counter } from "./Counter";

test("increments on click", async () => {
  const user = userEvent.setup();
  render(<Counter />);

  expect(screen.getByText("Count: 0")).toBeInTheDocument();

  await user.click(screen.getByRole("button", { name: /increment/i }));

  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});
```

### Testing async UI

```tsx
test("loads and displays user", async () => {
  render(<UserView id="42" />);

  expect(screen.getByRole("status")).toHaveTextContent(/loading/i);

  // findBy* waits up to 1s for the element to appear
  expect(await screen.findByRole("heading", { name: "Ana" })).toBeInTheDocument();
});
```

### MSW — mock the network, not your code

```ts
// mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users/:id", ({ params }) =>
    HttpResponse.json({ id: params.id, name: "Ana" }),
  ),
];
```

```ts
// test-setup.ts
import { setupServer } from "msw/node";
import { handlers } from "./mocks/handlers";

const server = setupServer(...handlers);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

### Custom hook testing

```tsx
import { renderHook, act } from "@testing-library/react";
import { useToggle } from "./useToggle";

test("toggle flips state", () => {
  const { result } = renderHook(() => useToggle(false));

  expect(result.current[0]).toBe(false);
  act(() => result.current[1]());
  expect(result.current[0]).toBe(true);
});
```

### Form interaction

```tsx
test("submit shows confirmation", async () => {
  const user = userEvent.setup();
  render(<ContactForm />);

  await user.type(screen.getByLabelText(/email/i), "a@b.com");
  await user.type(screen.getByLabelText(/message/i), "hello");
  await user.click(screen.getByRole("button", { name: /send/i }));

  expect(await screen.findByText(/sent!/i)).toBeInTheDocument();
});
```

---

## Common Patterns

```tsx
// Pattern: render with providers (router, query client, auth)
function renderWithProviders(ui: React.ReactElement) {
  const qc = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(
    <QueryClientProvider client={qc}>
      <BrowserRouter>{ui}</BrowserRouter>
    </QueryClientProvider>,
  );
}
```

```tsx
// Pattern: assert what a user CAN'T see
expect(screen.queryByText(/error/i)).not.toBeInTheDocument();
//          ^^^^^^^ getByX throws on miss; queryByX returns null
```

---

## Gotchas & Tips

- **Always `await user.X()`** — `user-event` v14+ is async even for clicks. Forgetting `await` causes flaky tests.
- **`screen` is your friend.** Skip the destructured render result; query via `screen` so refactors don't break.
- **Prefer `getByRole`.** It catches a11y bugs (missing labels, wrong roles) for free.
- **`findByX` waits**, `getByX` is sync, `queryByX` returns null on miss. Pick the right one.
- **Don't snapshot HTML.** Snapshots break on every styling change and discourage thinking. Use focused assertions.
- **Wrap state-changing async work in `act()`** — RTL usually does this for you, but raw store updates may not.
- **Don't test internal implementation.** "Did `useEffect` run?" is the wrong question. "Did the user see the result?" is the right one.
- **Mock the network with MSW**, not the components. Tests stay realistic and survive refactors.
- **For visual regression**, use Playwright + screenshots, not Jest snapshots.
- **Run E2E sparingly** — slow + flaky. Most behavior should be covered by component-level RTL tests.

---

## See Also

- [[06 - Custom Hooks]]
- [[19 - TypeScript with React]]
- [[17 - Monitoring and Errors]]
