---
tags: [react, advanced, nextjs, ssr, rsc]
aliases: [Next.js, App Router]
level: Advanced
---

# Next.js App Router

> **One-liner**: Next.js's **App Router** is the file-system-based routing + RSC framework — folders become routes, special files (`page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`) compose a route's UI, and Server Components are the default.

---

## Quick Reference

| File | Purpose |
|------|---------|
| `app/page.tsx` | UI for the route (`/`) |
| `app/layout.tsx` | Wrapping layout (persists across child routes) |
| `app/loading.tsx` | Suspense fallback for the segment |
| `app/error.tsx` | Error boundary (must be a client component) |
| `app/not-found.tsx` | 404 UI |
| `app/template.tsx` | Like layout but re-mounts on navigation |
| `app/route.ts` | Route handler (REST endpoint) |
| `app/[id]/page.tsx` | Dynamic segment → `params.id` |
| `app/(group)/page.tsx` | Route group (organize without affecting URL) |
| `app/@named/page.tsx` | Parallel route slot |
| `app/(.)foo/page.tsx` | Intercepting route (modal-over-route) |

| Caching knob | Effect |
|--------------|--------|
| `export const dynamic = "force-static"` | Force SSG/cache |
| `export const dynamic = "force-dynamic"` | Force per-request |
| `export const revalidate = N` | ISR window in seconds |
| `fetch(..., { cache: "no-store" })` | Bypass cache for this fetch |
| `fetch(..., { next: { tags: ["x"] } })` | Tag for `revalidateTag("x")` |

---

## Core Concept

The App Router (introduced in Next 13, default since 14) replaces Pages Router. Key shifts:

1. **Server Components by default.** A `page.tsx` is a Server Component unless marked `"use client"`. Most fetching, DB access, and non-interactive UI lives on the server.
2. **Nested layouts.** `app/dashboard/layout.tsx` wraps every route under `/dashboard/*` — the layout persists across navigations within that segment, with no remount.
3. **Special-file convention.** Each segment can opt into a loading skeleton, error boundary, and 404 by creating `loading.tsx`, `error.tsx`, `not-found.tsx`.
4. **Streaming + Suspense.** Slow data resolves in the background; fast HTML streams immediately.
5. **Server Actions.** Mutations live next to data, called directly from forms/clients ([[05 - Server Actions]]).
6. **Caching layers.** `fetch` is cached by default; `revalidatePath` / `revalidateTag` invalidate; static and dynamic are decided per-segment.

It's a lot. The mental model: **start every component on the server. Push `"use client"` to leaf components that need state, effects, or browser APIs.**

---

## Diagram

```mermaid
graph TD
    Root[app/layout.tsx]
    Root --> Page[app/page.tsx /]
    Root --> Dash[app/dashboard/layout.tsx]
    Dash --> DashIdx[app/dashboard/page.tsx /dashboard]
    Dash --> DashSettings[app/dashboard/settings/page.tsx /dashboard/settings]
    Dash --> DashSettingsLoading[app/dashboard/settings/loading.tsx]
    Dash --> DashSettingsError[app/dashboard/settings/error.tsx]
    Root --> Blog[app/blog/[slug]/page.tsx /blog/:slug]
```

---

## Syntax & API

### Folder structure

```
app/
├── layout.tsx              # root layout (HTML shell + Providers)
├── page.tsx                # /
├── loading.tsx             # global loading
├── error.tsx               # root error boundary
├── not-found.tsx           # 404
├── api/
│   └── users/route.ts      # GET/POST handlers (REST endpoints)
├── (marketing)/            # route group (URL stays /pricing, /about)
│   ├── pricing/page.tsx
│   └── about/page.tsx
└── dashboard/
    ├── layout.tsx          # wraps everything under /dashboard
    ├── page.tsx            # /dashboard
    └── settings/
        ├── page.tsx        # /dashboard/settings
        └── loading.tsx     # streaming fallback
```

### Root layout — required, defines `<html>`/`<body>`

```tsx
// app/layout.tsx
export const metadata = { title: "MyApp" };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### Page with server-side data

```tsx
// app/users/[id]/page.tsx
export default async function UserPage({ params }: { params: { id: string } }) {
  const user = await db.user.findUnique({ where: { id: params.id } });
  if (!user) notFound();      // routes to nearest not-found.tsx
  return <h1>{user.name}</h1>;
}
```

### Loading + error per segment

```tsx
// app/users/[id]/loading.tsx — auto-Suspense fallback
export default function Loading() { return <Skeleton />; }

// app/users/[id]/error.tsx — must be client + reset prop
"use client";
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

### Route handler (REST endpoint)

```ts
// app/api/users/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  const users = await db.user.findMany();
  return NextResponse.json(users);
}

export async function POST(req: Request) {
  const body = await req.json();
  const user = await db.user.create({ data: body });
  return NextResponse.json(user, { status: 201 });
}
```

### Dynamic vs static config

```tsx
// app/feed/page.tsx
export const revalidate = 60;                 // ISR — 60s
// or
export const dynamic = "force-dynamic";       // SSR per request
```

### Metadata + SEO

```tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Pricing",
  description: "Plans and pricing",
  openGraph: { images: ["/og.png"] },
};

// Dynamic metadata
export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPost(params.slug);
  return { title: post.title };
}
```

### Parallel + intercepting routes (modal-over-route)

```
app/
├── @modal/
│   └── (.)photos/[id]/page.tsx   # /photos/:id intercepts to render in @modal slot
└── photos/[id]/page.tsx          # full-page version (direct nav, refresh)
```

---

## Common Patterns

```tsx
// Pattern: keep server data on the server; pass to client island
export default async function Page() {
  const data = await db.query();
  return <ChartClient data={data} />;
}

"use client";
export function ChartClient({ data }: { data: Point[] }) {
  // visualization library lives only in client bundle
  return <Chart data={data} />;
}
```

```tsx
// Pattern: shared layout with persistent state (e.g., dashboard nav)
// app/dashboard/layout.tsx
export default function DashLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="grid grid-cols-[16rem_1fr]">
      <Sidebar />     {/* doesn't remount on /dashboard/* navigation */}
      <main>{children}</main>
    </div>
  );
}
```

---

## Gotchas & Tips

- **`page.tsx` is required** to make a folder routable. A folder with only `layout.tsx` is not a route.
- **Layouts don't re-render on navigation** (only their children's segment does). Use `template.tsx` if you need a fresh mount per nav.
- **Error boundaries must be client components** (`"use client"` at the top).
- **Don't put `useState`/`useEffect` in `page.tsx` or `layout.tsx`** — they're server components by default. Add `"use client"` or extract to a client child.
- **`fetch` is automatically deduplicated and cached** in server components — opt out with `cache: "no-store"`.
- **`cookies()`, `headers()`, `searchParams`** flip a route into dynamic mode (not cacheable). That's usually correct; sometimes surprising.
- **Use `notFound()` and `redirect()`** from `next/navigation` to bail out of a route.
- **Dynamic params arrive as strings.** Validate them.
- **Static export (`output: "export"`)** turns your app into pure HTML — no SSR, no actions, no route handlers.
- **Edge vs Node runtime** — set per-route. Edge is faster but limited APIs.

---

## See Also

- [[04 - Server Components]]
- [[05 - Server Actions]]
- [[07 - Rendering Strategies]]
- [[09 - Remix and React Router 7]]
