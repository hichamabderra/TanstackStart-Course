# Module 05 — Search Params Deep Dive: The URL Is Your State

> Search params are TanStack Router's most under-rated superpower. This module turns
> `/products?search=laptop&page=2&sort=price` into a fully typed, validated, shareable state machine.
> Capstone relevance: every list page (products, orders, users) is driven by this pattern.

---

## 5.1 Why URL state beats React state for list pages

| Property | `useState` | URL search params |
|---|---|---|
| Survives refresh / deep-link | ❌ | ✅ |
| Shareable & bookmarkable | ❌ | ✅ |
| Back/forward works | ❌ | ✅ |
| Server can read it during SSR | ❌ | ✅ (loaders see it!) |
| Caches can key on it | ❌ | ✅ (router cache + HTTP) |
| Works pre-hydration | ❌ | ✅ |

Rule of thumb: **if the user would be annoyed to lose it on refresh, it belongs in the URL.** Filters,
pagination, sorting, tabs, selected item, date ranges. Form *drafts* and ephemeral UI (open dropdown) stay
in React state.

## 5.2 Declare a search schema with `validateSearch` `[BOTH]`

FILE: `src/routes/products/index.tsx`

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

// [BOTH] — shared schema: used by the router AND by server functions (one source of truth)
export const productsSearchSchema = z
  .object({
    search: z.string().optional(),
    page: z.number().int().min(1).default(1),
    pageSize: z.number().int().min(10).max(100).default(20),
    sort: z.enum(['newest', 'price-asc', 'price-desc', 'name']).default('newest'),
    category: z.string().optional(),
    createdAfter: z.coerce.date().optional(),
  })
  .default({}) // every field has a default → schema can parse {}

export type ProductsSearch = z.infer<typeof productsSearchSchema>

export const Route = createFileRoute('/products/')({
  // Runs on BOTH server (SSR) and client (navigation) — keep it pure & fast.
  validateSearch: (raw: Record<string, unknown>): ProductsSearch =>
    productsSearchSchema.parse(raw),

  loaderDeps: ({ search }) => ({ ...search }),   // Module 06: loader re-runs when search changes
  loader: ({ deps }) => listProducts({ data: deps }),
  component: ProductsPage,
})
```

What just happened:

1. **Parsing** — the URL's `?page=2` arrives as a string; Zod coerces it to `2`.
2. **Validation** — `?sort=hacker` throws; pair with `errorComponent` or use `.catch()` per field for graceful degradation.
3. **Defaults** — visiting bare `/products` yields `{ page: 1, pageSize: 20, sort: 'newest' }`.
4. **Typing** — `Route.useSearch()` returns `ProductsSearch`; the loader's `deps` is typed; the server
   function's `.validator(productsSearchSchema)` reuses the *same schema* (Module 10). **One schema, three
   environments.** That is the golden loop.

### Graceful degradation for user-editable URLs

```ts
// [BOTH] — never 500 because someone typed page=-3
export const productsSearchSchema = z.object({
  page: z.coerce.number().int().min(1).catch(1),
  search: z.string().catch(undefined),
  sort: z.enum(['newest', 'price-asc']).catch('newest'),
}).default({})
```

## 5.3 Reading search state `[CLIENT]` (and `[SERVER]` during SSR)

```tsx
function ProductsPage() {
  const search = Route.useSearch()      // typed ProductsSearch
  const { page, sort, search: q } = search
  // …
}
```

## 5.4 Updating search state — functional, merged, typed `[CLIENT]`

```tsx
import { useNavigate, Link } from '@tanstack/react-router'

function FilterBar() {
  const navigate = useNavigate({ from: Route.fullPath })

  return (
    <>
      {/* Declarative: functional update — `prev` is fully typed */}
      <Link
        search={(prev) => ({ ...prev, page: 1, search: 'laptop' })}
        replace // don't pollute history on filter changes
      >
        Search laptops
      </Link>

      {/* Imperative (event handlers) */}
      <button onClick={() => navigate({ search: (prev) => ({ ...prev, page: prev.page + 1 }) })}>
        Next page
      </button>
    </>
  )
}
```

### Push vs replace — a real product decision

- **`replace: true`** for keystroke-level changes (typing in a search box, changing page size, tab
  switches) — back button stays meaningful.
- **Push (default)** for discrete navigation acts a user expects to go "back" from.
- Debounce free-text search (~300ms) before writing to the URL.

## 5.5 Custom serialization `[BOTH]`

Dates, arrays, and booleans serialize poorly by default — control it:

```ts
// [BOTH] — arrays: ?tag=a&tag=b
validateSearch: (raw) => tagsSearchSchema.parse({
  ...raw,
  tag: raw.tag ? (Array.isArray(raw.tag) ? raw.tag : [raw.tag]) : undefined,
})

// Dates: store as ISO strings in the URL, coerce to Date in the schema:
createdAfter: z.coerce.date().optional() // accepts '2026-09-01'
```

Keep URL encoding human-readable — the URL is a UI surface.

## 5.6 Wiring search → loader → server → DB (the full loop)

```
URL  /products?search=laptop&page=2&sort=price-asc
  ↓ validateSearch (parse + defaults)              [BOTH]
  ↓ loaderDeps: router watches these deps          [BOTH]
  ↓ loader re-runs when deps change                [BOTH]
  ↓ listProducts({ data: deps })                   [SERVER] (server function)
  ↓ zod re-validates input (defense in depth)      [SERVER]
  ↓ repository builds SQL: WHERE/ORDER BY/LIMIT    [SERVER]
  ↓ rows + total → serialized back                 [SERVER → CLIENT]
  ↓ UI renders list + pagination controls          [CLIENT]
```

When the user clicks “Next page”, only the changed loader re-runs (router cache, Module 07); the rest of the
page doesn't remount.

## 5.7 Realistic example: tabs + filters + pagination together

```tsx
// [CLIENT] — src/routes/orders/-components/orders-toolbar.tsx
function OrdersToolbar() {
  const navigate = useNavigate({ from: '/orders' })
  const set = (patch: Partial<OrdersSearch>) =>
    navigate({ search: (prev) => ({ ...prev, ...patch, page: 1 }), replace: true })

  return (
    <div role="toolbar" className="flex gap-2">
      {/* Tabs are search state too */}
      {(['all', 'pending', 'shipped'] as const).map((status) => (
        <Link key={status} to="/orders" search={(p) => ({ ...p, status, page: 1 })} replace
              activeProps={{ 'aria-pressed': true }}>
          {status}
        </Link>
      ))}
      <input aria-label="Search orders" onChange={(e) => set({ q: e.target.value })} />
    </div>
  )
}
```

Module 20 binds this same state to TanStack Table sorting; Module 22 shows Query prefetching per search state.

## 5.8 Anti-patterns

⚠️ **DO NOT** mirror URL state into `useState` “for convenience” — you now have two sources of truth and a
sync bug. Read from the URL; write to the URL.

⚠️ **DO NOT** put secrets or sensitive ids you wouldn't show in a screenshot into search params (they're in
history, logs, referrers).

⚠️ **DO NOT** make `validateSearch` async or slow — it runs on every navigation. Pure + fast.

⚠️ **DO NOT** skip server-side re-validation (the server function's own `.validator`) just because the router
validated — the URL is user input, and server functions are network endpoints (Module 25).

## 5.9 Exercises

- **Beginner:** Implement `/products` with `search`, `page`, `sort` per 5.2; render the current parsed state as JSON on the page. Break the URL by hand (`?page=-9`, `?sort=zzz`) and watch graceful degradation.
- **Intermediate:** Add `createdAfter`/`createdBefore` date-range filters with ISO serialization, and make the back button replay a user's exact filter journey.
- **Production:** Add a “Copy link to current filters” button and server-render the correct page on first request (verify: View Source shows page-2 items).
- **Debug Challenge:** A teammate adds `search` state via `useState` and complains the SSR page always shows page 1 while the client shows page 3. Explain the bug using this module's model.

🧠 **MENTAL MODEL — Search params:** the query string is a *validated, typed, shareable slice of application
state* owned by the router. Components are projections of it; the URL is the store.

---

**Next: [Module 06 — Route Loaders & beforeLoad: Route-Centric Data Loading →](06-loaders.md)**
