---
tags: [react, advanced, remix, routing, ssr]
aliases: [Remix, React Router 7]
level: Advanced
---

# Remix and React Router 7

> **One-liner**: Remix and React Router 7 are essentially the same framework now (the merge happened in 2024) — file-system routing, **loaders** for reads, **actions** for writes, web-standard `Request`/`Response`/`<form>` for everything else, with optional SSR and emerging RSC support.

---

## Quick Reference

| Feature | Where it lives |
|---------|----------------|
| Route file | `app/routes/foo.tsx` (or v7 file convention) |
| Read data | `export async function loader({ params, request }) { ... }` |
| Write data | `export async function action({ request, params }) { ... }` |
| Render | `export default function Component()` (uses `useLoaderData`) |
| Navigate | `<Link to>` / `useNavigate` |
| Submit | `<Form method="post">` / `useFetcher` (no full reload, but works without JS) |
| Error UI | `export function ErrorBoundary()` |
| Streaming | `defer({ slow: slowPromise })` + `<Await resolve={...}>` |
| Pending UI | `useNavigation().state` |
| Cache | None built in — leverage HTTP `Cache-Control` headers |

---

## Core Concept

Remix's pitch: **stop reinventing what the browser already does**. Forms, links, redirects, HTTP caching, status codes — they're all web standards. Remix routes expose a `loader` (GET handler), an `action` (POST/PATCH/DELETE handler), and a default-export Component. The framework wires `<Form>` and `<Link>` to those, falls back to native browser behavior when JS isn't available, and uses fetch when it is.

In **2024**, Remix merged into **React Router 7**. They're the same project now: React Router gets Remix's framework features (loaders, actions, SSR, file routing, build pipeline); Remix users get React Router's broader ecosystem. New apps can pick either name — the docs converge.

Compared to Next.js App Router:
- **Less magic**, smaller surface area.
- **Native web platform first** (Request/Response objects, FormData, `Set-Cookie`).
- **No built-in cache layer**; you set HTTP headers.
- **RSC** is opt-in (still maturing in late 2025); Next made RSC the default.
- **Nested routing with parallel data loading** is core; layouts share the same model.

---

## Syntax & API

### Project setup

```bash
npx create-react-router@latest my-app
cd my-app
npm run dev
```

### Route — loader, action, component, error boundary

```tsx
// app/routes/users.$id.tsx (or app/routes/users/[id].tsx in v7)
import type { Route } from "./+types/users.$id";
import { Form, useLoaderData, useNavigation } from "react-router";

export async function loader({ params }: Route.LoaderArgs) {
  const user = await db.user.findUnique({ where: { id: params.id } });
  if (!user) throw new Response("Not found", { status: 404 });
  return { user };
}

export async function action({ request, params }: Route.ActionArgs) {
  const formData = await request.formData();
  const name = formData.get("name");
  if (typeof name !== "string" || !name) {
    return { fieldErrors: { name: "Required" } };
  }
  await db.user.update({ where: { id: params.id }, data: { name } });
  return { ok: true };
}

export default function UserRoute() {
  const { user } = useLoaderData<typeof loader>();
  const nav = useNavigation();
  const submitting = nav.state === "submitting";

  return (
    <>
      <h1>{user.name}</h1>
      <Form method="post">
        <input name="name" defaultValue={user.name} />
        <button type="submit" disabled={submitting}>
          {submitting ? "Saving…" : "Save"}
        </button>
      </Form>
    </>
  );
}

export function ErrorBoundary({ error }: { error: unknown }) {
  return <p>Oops — {String(error)}</p>;
}
```

### Streaming with `defer` + `<Await>`

```tsx
import { defer } from "react-router";
import { Await, useLoaderData } from "react-router";
import { Suspense } from "react";

export async function loader() {
  return defer({
    user: await fetchUserFast(),       // blocks (immediate)
    feed: fetchFeedSlow(),             // streams in (promise)
  });
}

export default function Page() {
  const { user, feed } = useLoaderData<typeof loader>();
  return (
    <>
      <h1>Hi, {user.name}</h1>
      <Suspense fallback={<FeedSkeleton />}>
        <Await resolve={feed}>
          {(items: Item[]) => <Feed items={items} />}
        </Await>
      </Suspense>
    </>
  );
}
```

### Programmatic mutation with `useFetcher`

```tsx
import { useFetcher } from "react-router";

function ToggleStar({ itemId }: { itemId: string }) {
  const fetcher = useFetcher();

  return (
    <fetcher.Form method="post" action={`/items/${itemId}/star`}>
      <button name="op" value="toggle">
        {fetcher.state === "submitting" ? "…" : "★"}
      </button>
    </fetcher.Form>
  );
}
```

### Redirect from action

```tsx
import { redirect } from "react-router";

export async function action({ request }: Route.ActionArgs) {
  await login(await request.formData());
  return redirect("/dashboard");
}
```

---

## Common Patterns

```tsx
// Pattern: cookie-based session
import { createCookieSessionStorage } from "react-router";
const session = createCookieSessionStorage({ cookie: { name: "_sess", secrets: [SECRET] } });

export async function loader({ request }: Route.LoaderArgs) {
  const s = await session.getSession(request.headers.get("Cookie"));
  if (!s.get("userId")) throw redirect("/login");
  return { userId: s.get("userId") };
}
```

```tsx
// Pattern: HTTP caching via headers (no framework cache needed)
export function headers() {
  return { "Cache-Control": "public, max-age=60, s-maxage=300" };
}
```

```tsx
// Pattern: optimistic UI with useFetcher
const fetcher = useFetcher();
const optimisticName = fetcher.formData?.get("name") ?? user.name;
return <h1>{optimisticName}</h1>;
```

---

## Gotchas & Tips

- **Loaders run on every navigation** — but the framework dedupes parallel routes that haven't changed.
- **Throwing `Response` works as a redirect/error.** Throw a `404` `Response` from a loader to render the nearest `ErrorBoundary`.
- **`Form` (capital F) is the framework component.** `<form>` (lowercase) is the native element — it'll do a full reload.
- **`useFetcher` is for non-navigation submits** (button-on-card, autosave) — same loader/action API, no URL change.
- **Actions are typed-revalidated** — after an action returns, all loaders in the matched route tree refetch automatically.
- **No global cache by default.** Use `Cache-Control` headers and CDN; or wrap with TanStack Query for client-side caching of fetcher data.
- **Migrating from Remix → React Router 7** is mostly an import swap (`@remix-run/*` → `react-router`) and small config changes.
- **RSC support is rolling out** in v7+; until stable, you write traditional SSR with loaders/actions.
- **No file conventions for `loading.tsx` like Next** — use `useNavigation().state === "loading"` for global pending UI.

---

## See Also

- [[13 - React Router]]
- [[08 - Next.js App Router]]
- [[05 - Server Actions]]
- [[03 - Suspense]]
