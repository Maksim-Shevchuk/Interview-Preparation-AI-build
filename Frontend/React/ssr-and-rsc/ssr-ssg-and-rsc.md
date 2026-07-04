# SSR, SSG, and React Server Components

Rendering strategies define **where and when** your React app generates HTML. This is a high-signal interview topic
for full-stack positions, especially with Next.js experience.

## Rendering Strategies Overview

```
               Build time          Server (per request)         Client (browser)
               ──────────          ────────────────────         ────────────────
  SSG          HTML generated      Serve static files           Hydrate + interactive
  ISR          HTML generated      Revalidate in background     Hydrate + interactive
  SSR          —                   HTML generated               Hydrate + interactive
  CSR          —                   Serve empty HTML + JS        Render everything
  RSC          —                   Render server components     Hydrate client components only
```

## Client-Side Rendering (CSR)

Traditional SPA approach (Create React App, Vite):

```
Browser receives:  <div id="root"></div> + large JS bundle
                   ↓
                   JS downloads, parses, executes
                   ↓
                   React renders the entire UI in the browser
```

**Pros:** Simple deployment (static hosting), rich interactivity.
**Cons:** Slow initial load (blank screen until JS loads), poor SEO (empty HTML), large bundle.

## Server-Side Rendering (SSR)

HTML is generated **on the server for each request**:

```
Browser requests page → Server runs React → Returns full HTML → Browser hydrates
```

```tsx
// Next.js (App Router)
// By default, components in app/ are Server Components (rendered on server)

// Force dynamic SSR (no caching)
export const dynamic = "force-dynamic";

export default async function UsersPage() {
    const users = await db.users.findMany(); // runs on server
    return <UserList users={users} />;
}
```

**Hydration** — the process where React attaches event listeners and makes the server-rendered HTML interactive. React
"adopts" the existing DOM instead of re-creating it.

**Pros:** Fast First Contentful Paint (FCP), good SEO, works without client JS.
**Cons:** Slower Time to First Byte (TTFB) — server must render on each request. Full page JS still needed for
hydration.

### Streaming SSR (React 18+)

Server sends HTML in chunks as components resolve — the browser can start rendering before the full page is ready:

```tsx
<Suspense fallback={<Skeleton />}>
    <SlowComponent />   {/* streamed in when ready */}
</Suspense>
```

Benefits: faster perceived load, no all-or-nothing blocking.

## Static Site Generation (SSG)

HTML is generated **at build time**:

```tsx
// Next.js — pages with no dynamic data are automatically SSG
export default function AboutPage() {
    return <div>About us</div>;
}

// With data fetching at build time
export async function generateStaticParams() {
    const posts = await getPosts();
    return posts.map(post => ({ slug: post.slug }));
}
```

**Pros:** Fastest possible load (pre-built HTML served from CDN), cheapest hosting.
**Cons:** Data can become stale, rebuild required for updates, not suitable for personalized content.

## Incremental Static Regeneration (ISR)

Combines SSG with on-demand revalidation — pages are statically generated but **refreshed in the background**:

```tsx
// Next.js App Router
export const revalidate = 60; // revalidate every 60 seconds

export default async function ProductsPage() {
    const products = await getProducts();
    return <ProductList products={products} />;
}
```

**How it works:**
1. First request: serve the static page.
2. After `revalidate` seconds: next request triggers a background regeneration.
3. Subsequent requests get the fresh page.

## React Server Components (RSC)

A fundamentally new model (React 18+, stable in Next.js App Router). Components are split into **Server Components**
and **Client Components**.

### Server Components (default in Next.js `app/`)

- Run **only on the server** — never shipped to the client bundle.
- Can directly access databases, file system, internal APIs.
- Cannot use hooks (`useState`, `useEffect`) or browser APIs.
- Cannot have event handlers (`onClick`, `onChange`).
- Output is a **serialized React tree** (not HTML) streamed to the client.

```tsx
// app/users/page.tsx — Server Component by default
export default async function UsersPage() {
    const users = await db.users.findMany(); // direct DB access — no API layer needed

    return (
        <div>
            <h1>Users</h1>
            {users.map(user => (
                <UserCard key={user.id} user={user} />  // can be server or client
            ))}
            <AddUserButton />  {/* must be client — has onClick */}
        </div>
    );
}
```

### Client Components (`"use client"`)

- Traditional React components — run in the browser.
- Can use hooks, event handlers, browser APIs.
- The `"use client"` directive marks the **boundary** — everything imported by a client component is also client.

```tsx
"use client";

import { useState } from "react";

export function AddUserButton() {
    const [isOpen, setIsOpen] = useState(false);

    return (
        <>
            <button onClick={() => setIsOpen(true)}>Add User</button>
            {isOpen && <Modal onClose={() => setIsOpen(false)} />}
        </>
    );
}
```

### Server vs Client Component Rules

| Feature                    | Server Component | Client Component |
|----------------------------|:---:|:---:|
| `async`/`await` in component | ✅ | ❌ |
| Direct DB/filesystem access  | ✅ | ❌ |
| `useState`, `useEffect`      | ❌ | ✅ |
| Event handlers (`onClick`)   | ❌ | ✅ |
| Browser APIs (`window`, `localStorage`) | ❌ | ✅ |
| Included in JS bundle        | ❌ | ✅ |
| Can render client components  | ✅ | ✅ |
| Can render server components  | ✅ | ❌ (can accept as `children`) |

### Composition Pattern

Server components can **pass** server-rendered content to client components via `children`:

```tsx
// Server Component
export default async function Page() {
    const data = await fetchData(); // server-only
    return (
        <ClientWrapper>
            <ServerRenderedContent data={data} />  {/* rendered on server, passed as children */}
        </ClientWrapper>
    );
}

// Client Component
"use client";
export function ClientWrapper({ children }: { children: React.ReactNode }) {
    const [isOpen, setIsOpen] = useState(true);
    return isOpen ? <div>{children}</div> : null;
}
```

### Benefits of RSC

1. **Zero bundle size for server components** — DB queries, heavy libraries (markdown parsers, syntax highlighters)
   stay on the server.
2. **Direct backend access** — no need for API routes to fetch data.
3. **Automatic code splitting** — client components are only loaded when needed.
4. **Streaming** — server components can `await` data and stream the result.

## Next.js App Router Summary

| File convention      | Purpose                                      |
|----------------------|----------------------------------------------|
| `page.tsx`           | Route component (Server Component by default)|
| `layout.tsx`         | Shared layout (wraps child routes)           |
| `loading.tsx`        | Loading UI (Suspense boundary)               |
| `error.tsx`          | Error UI (Error Boundary) — must be `"use client"` |
| `not-found.tsx`      | 404 page                                     |
| `route.ts`           | API route handler (GET, POST, etc.)          |

## Common Interview Questions

1. **CSR vs SSR vs SSG?** — CSR: rendered in browser (slow initial load, good interactivity). SSR: rendered on
   server per request (fast FCP, good SEO). SSG: rendered at build time (fastest, but stale data).
2. **What is hydration?** — The process where React attaches event listeners to server-rendered HTML, making it
   interactive. React reuses the existing DOM nodes.
3. **What are React Server Components?** — Components that run only on the server, aren't included in the client
   bundle, and can directly access backend resources. They cannot use hooks or event handlers.
4. **`"use client"` — what does it do?** — Marks a module as the boundary between server and client. The component
   and everything it imports is included in the client bundle.
5. **Can a client component render a server component?** — Not directly (it can't import one). But a server
   component can pass server-rendered content to a client component via `children` prop.
6. **What is streaming SSR?** — Server sends HTML in chunks using `Suspense` boundaries. The browser renders content
   progressively as it arrives, without waiting for the full page.

## Related

- [Components, JSX, and Virtual DOM](../fundamentals/components-jsx-and-virtual-dom.md) — reconciliation, Fiber
- [Performance Optimization](../performance/performance-optimization.md) — code splitting, lazy loading
- [State Management](../state-management/state-management.md) — server state with React Query

## Resources

- [React Docs — Server Components](https://react.dev/reference/rsc/server-components)
- [Next.js Docs — App Router](https://nextjs.org/docs/app)
- [Dan Abramov — The Two Reacts](https://overreacted.io/the-two-reacts/)
- [Vercel — Understanding React Server Components](https://vercel.com/blog/understanding-react-server-components)
