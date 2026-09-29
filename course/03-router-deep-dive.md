# Module 03 — TanStack Router Deep Dive: Route Trees & File-Based Routing

> TanStack Start is built on TanStack Router. If your router knowledge is shallow, your Start knowledge is
> shallow. This module goes deep — but stays on **stable v1 APIs** (verified current in Module 00).

---

## 3.1 The core idea: a route TREE, not a route list

Most routers are flat tables: `path → component`. TanStack Router models routes as a **tree**, mirroring the
URL segments:

```
/                       __root.tsx            (document shell + global layout)
├── /products           products.tsx          (layout: list shell, filters sidebar)
│   └── /products/$id   products.$id.tsx      (detail; renders INSIDE products.tsx's <Outlet/>)
├── /dashboard          dashboard.tsx
│   ├── /dashboard/orders …
│   └── /dashboard/analytics …
└── /login              login.tsx
```

Why a tree? Because **real UIs are trees of layouts**: the product detail page shares the products layout,
which shares the app shell. Each level can own: a component (layout), a loader, `beforeLoad`, search schema,
error boundary, pending UI, and context. Navigation between sibling leaves re-runs only what changed.

### The pipeline (this is the WHOLE feature)

```
filesystem (src/routes/*)
   ↓  Vite plugin scans + generates (dev + build)
routeTree.gen.ts  (generated, typed, DO NOT EDIT)
   ↓  imported by router.tsx
createRouter({ routeTree })
   ↓  TypeScript infers every path/param/search/loader type
typed APIs: <Link to="/products/$id" params={{…}} /> · navigate() · useLoaderData()
   ↓  at runtime
route matching + parallel loaders + rendering
```

**One file move = URL change = type change = compiler feedback everywhere.** That feedback loop is the product.

## 3.2 File conventions (verified against the Router docs)

FILE: `src/routes/products.tsx` `[SERVER → CLIENT]`

```tsx
import { createFileRoute, Link, Outlet } from '@tanstack/react-router'

export const Route = createFileRoute('/products')({
  component: ProductsLayout,
})

function ProductsLayout() {
  return (
    <div className="grid md:grid-cols-[240px_1fr] gap-6">
      <aside>{/* filters go here in Module 05 */}</aside>
      <main><Outlet /></main> {/* children render here */}
    </div>
  )
}
```

FILE: `src/routes/products/$productId.tsx` `[SERVER → CLIENT]`

```tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/products/$productId')({
  component: ProductDetail,
})

function ProductDetail() {
  const { productId } = Route.useParams() // fully typed: { productId: string }
  return <h1>Product {productId}</h1>
}
```

### Naming rules

| File | Route path | Notes |
|---|---|---|
| `__root.tsx` | (root, matches everything) | Document shell |
| `index.tsx` | `/` | Index route |
| `about.tsx` | `/about` | Static segment |
| `products.tsx` + `products/index.tsx` | `/products` layout + `/products` index | Layout pairing |
| `products/$productId.tsx` | `/products/:productId` | Dynamic param (`$` prefix) |
| `files/$.tsx` | `/files/*` | **Splat / catch-all** (matched via `Route.useParams()._splat`) |
| `_authed.tsx` + `_authed/dashboard.tsx` | `/dashboard` | **Pathless layout** (`_` prefix: layout without URL segment) |
| `users[.]json.ts` | `/users.json` | Escaped literal dot |
| `my-page.tsx` vs `my_page.tsx` | `/my-page` vs `/my_page` | `_` in the middle is literal; use `-` for words |

**Route groups / folder organization:** to keep `src/routes/` tidy, put related routes in folders —
the *path* is declared in `createFileRoute('...')`, so folders are organizational, not structural.
Folders prefixed with `_` are ignored for URL purposes.

## 3.3 Navigation

FILE: `src/routes/products/index.tsx` `[BOTH]`

```tsx
import { Link, useNavigate } from '@tanstack/react-router'

export function ProductCards() {
  const navigate = useNavigate()
  return (
    <>
      {/* Declarative — preferred. Prefetchable, accessible, middle-click works. */}
      <Link
        to="/products/$productId"
        params={{ productId: '42' }}
        search={{ sort: 'price' }}           // typed search (Module 05)
        activeProps={{ className: 'font-bold' }}
      >
        View product
      </Link>

      {/* Imperative — for after mutations, redirects in handlers, etc. */}
      <button onClick={() => navigate({ to: '/login', search: { redirect: '/products' } })}>
        Sign in
      </button>
    </>
  )
}
```

> ⚠️ **DO NOT** use raw `<a href>` for internal navigation — you lose client-side routing, prefetching, and
> type checking. `<a>` is for *external* URLs only.

## 3.4 Route option tour (the parts Start builds on)

```tsx
export const Route = createFileRoute('/dashboard')({
  // — data & guards (Module 06) —
  beforeLoad: async ({ params, search, context }) => { /* auth/redirect UX */ },
  loaderDeps: ({ search }) => ({ page: search.page }),
  loader: async ({ deps, context }) => { /* fetch data */ },

  // — UI —
  component: DashboardPage,
  pendingComponent: DashboardSkeleton,   // shown while THIS route loads
  errorComponent: ({ error, reset }) => <ErrorBox error={error} reset={reset} />,
  notFoundComponent: () => <p>Nothing here.</p>,

  // — UX tuning —
  defaultPendingMs: 300,     // wait this long before showing pending UI (avoid flash)
  defaultPendingMinMs: 500,  // once shown, keep it at least this long (avoid flicker)

  // — caching (Module 07) —
  staleTime: 30_000,
  gcTime: 5 * 60_000,

  // — search schema (Module 05) —
  validateSearch: (raw): ProductSearch => productsSearchSchema.parse(raw),
})
```

### Redirects & not-found — thrown, not returned `[BOTH]`

```tsx
import { redirect, notFound } from '@tanstack/react-router'

beforeLoad: async ({ context }) => {
  if (!context.session) throw redirect({ to: '/login' })
},
loader: async ({ params }) => {
  const order = await getOrder({ data: { id: params.orderId } })
  if (!order) throw notFound()
  return order
},
```

### Error boundaries are per-route `[SERVER → CLIENT]`

Any route can define `errorComponent` / `notFoundComponent`; the router catches loader/render errors at the
**deepest route that defines a boundary**, so a broken widget doesn't nuke the whole app. The root route
should always define a last-resort `errorComponent`.

## 3.5 Code splitting with lazy routes `[CLIENT-leaning]`

Split route *options* (heavy components, charting libs) out of the critical bundle:

FILE: `src/routes/dashboard/analytics.lazy.tsx`

```tsx
import { createLazyFileRoute } from '@tanstack/react-router'

export const Route = createLazyFileRoute('/dashboard/analytics')({
  component: AnalyticsPage, // loaded on demand
})
```

The base route (without `.lazy`) keeps `loader`/`beforeLoad` in the main bundle — **data loads immediately,
pixels arrive later**. This pattern is what makes preload + streaming feel instant (Modules 07/09).

## 3.6 Code-based routing: the alternative `[BOTH]`

You *can* build the tree manually:

```tsx
import { createRootRoute, createRoute, createRouter } from '@tanstack/react-router'

const rootRoute = createRootRoute({ component: Shell })
const productsRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: 'products',
  component: Products,
})
const routeTree = rootRoute.addChildren([productsRoute])
```

**When it makes sense:** monorepo-generated routes, plugin systems, dynamic route composition in libraries.
**Default choice for apps:** file-based — the generator gives you the tree, the types, and the code-splitting
conventions for free. The course uses file-based throughout.

## 3.7 Worked example: Meridian route skeleton (Capstone Stage 2)

```
src/routes/
├── __root.tsx                 # shell, nav, theme, error boundary
├── index.tsx                  # landing (public)
├── pricing.tsx                # public
├── login.tsx / register.tsx   # auth
├── _authed.tsx                # pathless layout: session guard via beforeLoad
├── _authed/
│   ├── dashboard.tsx          # streaming dashboard (Module 09)
│   ├── products.tsx           # list + search params (Module 05)
│   ├── products.$productId.tsx
│   ├── orders.tsx
│   └── settings.tsx
├── _admin.tsx                 # pathless layout: role guard (Module 15)
├── _admin/
│   ├── admin.users.tsx        # TanStack Table (Module 20)
│   ├── admin.products.tsx
│   └── admin.analytics.tsx
└── api/
    ├── auth.$.ts              # Better Auth mount (Module 14) — SERVER ROUTE
    └── webhooks.stripe.ts     # SERVER ROUTE (Module 11)
```

Notice: **server routes live in the same tree.** `/api/auth/$` is a catch-all *server* route; `_authed` is a
*pathless client layout*. One tree, both worlds.

## 3.8 Exercises

- **Beginner:** Build the route skeleton above (components can be stubs). Verify: visiting `/dashboard` while logged-out redirects (hardcode for now); `/products/42` renders inside the products layout.
- **Intermediate:** Add `pendingComponent` + `defaultPendingMs: 500` to a route with an artificial 1s loader delay. Watch the anti-flicker behavior.
- **Production:** Make the root `errorComponent` report to a logging endpoint (server route!) without crashing the app.
- **Debug Challenge:** You rename `products.$productId.tsx` → `products.$id.tsx` but forget to update a `<Link params={{ productId }} />`. What fails, and *when* do you find out? (This is the point of typed routing — answer in Module 04.)

🧠 **MENTAL MODEL — Router:** the route tree is a *typed, generated, shared* data structure. Files are the
source of truth; the generator makes the tree; the compiler checks every consumer; the runtime matches,
loads, and renders. Navigation is just a state change on that tree.

---

**Next: [Module 04 — Type-Safe Routing: the Compiler as Co-Pilot →](04-type-safe-routing.md)**
