# Module 04 — Type-Safe Routing: The Compiler as Your Co-Pilot

> This is TanStack Router's flagship feature and the reason large teams choose it. We don't just show the
> types — we show what they **catch**, because that's the value you're buying.

---

## 4.1 What is actually typed

Because `routeTree.gen.ts` registers every route's path, params, search schema, loader return, and context,
TypeScript knows the *entire* URL space of your app. Everything below is checked at compile time:

| Surface | What's checked |
|---|---|
| `to` / `from` props | Route id must exist in the tree |
| `params` | Exact params object for that route (`$productId` → `{ productId: string }`) |
| `search` | Matches the route's `validateSearch` schema (incl. defaults) |
| `Link` autocomplete | `to` suggestions come from your actual files |
| `Route.useParams()` / `useSearch()` / `useLoaderData()` | Return types inferred per route |
| Route context | What parents provide vs what children consume |
| `redirect({ to })`, `navigate({ to })` | Same guarantees as `Link` |
| Server function inputs/outputs | Serialization-checked by default (Module 10) |

## 4.2 Four bugs the compiler kills

### Bug 1 — Invalid route

```tsx
// ❌ Compile error: "/prodcts" is not a route id
<Link to="/prodcts" />
// ✅
<Link to="/products" />
```

Rename a route file → the route id changes → **every link, redirect, and navigate in the repo is flagged**.
No 404s discovered in production.

### Bug 2 — Missing/wrong params

```tsx
// ❌ Property 'id' does not exist — this route expects 'productId'
<Link to="/products/$productId" params={{ id: product.id }} />

// ❌ Missing params entirely
<Link to="/products/$productId" />
// ✅
<Link to="/products/$productId" params={{ productId: product.id }} />
```

### Bug 3 — Invalid search params (with Module 05's schema)

```tsx
// schema says: { search?: string, page: number, sort: 'price'|'name' }
// ❌ 'pric' is not assignable to 'price' | 'name'
<Link to="/products" search={{ page: 2, sort: 'pric' }} />

// ❌ page must be a number — URL strings are parsed/validated for you
<Link to="/products" search={{ page: '2' }} />
```

### Bug 4 — Misusing loader data

```tsx
export const Route = createFileRoute('/products/$productId')({
  loader: async ({ params }) => getProduct({ data: { id: params.productId } }),
  component: Page,
})

function Page() {
  const product = Route.useLoaderData()
  // ❌ Property 'prize' does not exist on type 'Product'
  return <p>{product.prize}</p>
  // ✅
  return <p>{product.price}</p>
}
```

## 4.3 How the typing works under the hood

1. `createFileRoute('/products/$productId')({…})` is a **curried call**: the first call fixes the path
   literal, the second checks your options against that path (so `loader` sees `params.productId: string`).
2. `routeTree.gen.ts` registers a big union of route ids and their meta into the router.
3. `declare module '@tanstack/react-router' { interface Register { router: … } }` (from Module 02) wires
   **your** router into hooks like `useNavigate()` globally — no prop drilling of the router type.
4. `Link` is generic over `to`/`search`/`params`, so JSX usage is fully checked.

You never hand-write these types; you get them by *using the APIs the way the docs show*.

## 4.4 Typed route context (preview) `[BOTH]`

Context flows down the route tree and is typed at each level (deep dive in Module 06):

```tsx
// parent provides:
export const Route = createFileRoute('/_admin')({
  beforeLoad: async () => ({ adminUser }),   // typed contribution
})

// child consumes — autocomplete knows about adminUser:
export const Route = createFileRoute('/_admin/users')({
  loader: async ({ context }) => listUsers({ data: { orgId: context.adminUser.orgId } }),
})
```

## 4.5 Why this matters at scale

> **WHY TANSTACK?** In a 200-route app, URLs are a *database schema for navigation*. Stringly-typed routers
> treat a route rename like a find-and-replace prayer. TanStack Router treats it like a schema migration:
> the compiler enumerates every consumer and blocks the merge until they're fixed. Teams report entire bug
> classes (dead links, wrong param names, search-param drift) simply stop existing.

Combine with Module 10: server functions are typed across the client/server boundary too. **URL → route →
loader → server function → database → component** is one continuous type. That is the actual pitch.

## 4.6 Exercises

- **Beginner:** In your skeleton, misspell a `to` prop and a `params` key; read the compiler messages aloud. Learn their shape — you'll read them weekly.
- **Intermediate:** Rename `products.$productId.tsx` to `products.$slug.tsx`. List every file the compiler flags. Then revert.
- **Production:** Add a `RouteId`-typed utility `goTo(routeId, params?)` wrapper used by all imperative navigation in your app; prove it keeps call sites checked.
- **Architecture Challenge:** Your PM wants "SEO-friendly URL change: `/products/$productId` → `/p/$productId`". In a stringly-typed router this is a sprint. With typed routing, enumerate the steps and the risks that *remain* after the compiler is happy. (Hint: external links, bookmarks — the compiler can't see outside the repo.)

🧠 **MENTAL MODEL — Type-safe routing:** route files *are* the schema; the generator *is* the migration tool;
the compiler *is* the integration test that runs on every keystroke.

---

**Next: [Module 05 — Search Params Deep Dive: The URL Is Your State →](05-search-params.md)**
