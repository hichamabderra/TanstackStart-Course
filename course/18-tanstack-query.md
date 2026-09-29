# Module 18 — TanStack Query in Start: Cache Architecture Decision Day

> One of the two most important architecture decisions in the course. We do NOT say “always use Query”.
> We teach you to *decide*, then integrate the official way (pattern verified against TanStack's
> `start-basic-react-query` example and the Router Query integration docs).

---

## 18.1 Router loader cache vs TanStack Query — the decision framework

| Dimension | Router loader cache (Module 07) | TanStack Query |
|---|---|---|
| Cache key | Route match (path+params+loaderDeps) | Arbitrary `queryKey` — by **entity**, shared across routes |
| Best shape | “Data belongs to this URL” | “Data is used by many components/routes” |
| Mutations | Manual `router.invalidate()` | `useMutation` lifecycle: pending/error/success/optimistic built-in |
| Shared UI consumers | Awkward (header badge + page + modal) | Natural: same key, many subscribers |
| Polling / refetch intervals | Not its job | `refetchInterval`, window-focus refetch |
| Infinite lists | Not its job | `useInfiniteQuery` |
| Offline persistence | No | Persisters exist |
| SSR integration | Automatic | `dehydrate`/`hydrate` (18.2) |
| Complexity cost | ~zero | QueryClient lifecycle to manage |

### Choose the ROUTER CACHE when

- Data is **route-specific** (the product detail page owns “the product”).
- Small/medium app; data naturally loads with routes.
- You want minimum machinery. **Start's default should be the router cache.**

### Choose TANSTACK QUERY when

- Data is **shared widely** (cart in header + page + checkout).
- Complex **mutation lifecycles** with optimistic updates and rollbacks.
- **Polling** (live dashboards), **infinite scroll**, dependent queries.
- Client components own the data lifecycle (e.g. a widget with its own refresh button).
- Offline scenarios.

### Both together — the mature pattern

Loaders call `queryClient.ensureQueryData(...)` → the *route* still decides WHEN data loads (SSR, preload,
cache policy), while *Query* owns the entity cache and mutation lifecycle. That's exactly what 18.2 wires.

## 18.2 The official integration: QueryClient + dehydrate/hydrate `[BOTH]`

FILE: `src/router.tsx`

```tsx
import { createRouter } from '@tanstack/react-router'
import { QueryClient, dehydrate, hydrate } from '@tanstack/react-query'
import { routeTree } from './routeTree.gen'

export function getRouter() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { staleTime: 30_000, gcTime: 5 * 60_000 }, // tune per app
    },
  })

  return createRouter({
    routeTree,
    context: { queryClient },                 // available to every loader via route context
    scrollRestoration: true,
    defaultPreload: 'intent',
    // SSR ↔ client cache transfer:
    dehydrate: () => dehydrate(queryClient),  // server: serialize Query cache into the document
    hydrate: (dehydrated) => hydrate(queryClient, dehydrated), // client: re-seed the cache
  })
}
```

FILE: `src/routes/__root.tsx` (provider around the tree)

```tsx
import { QueryClientProvider } from '@tanstack/react-query'
import { createRootRouteWithContext, Outlet } from '@tanstack/react-router'

export const Route = createRootRouteWithContext<{ queryClient: QueryClient }>()({
  component: () => {
    const { queryClient } = Route.useRouteContext()
    return (
      <QueryClientProvider client={queryClient}>
        <Outlet />
      </QueryClientProvider>
    )
  },
  // …head(), errorComponent from Module 02/08
})
```

> ⚠️ **DO NOT** create the QueryClient inside a component or per render — you'll reset the cache (classic
> bug from the debugging gym). One client per router instance.

## 18.3 Query keys are your entity API

FILE: `src/features/products/query-keys.ts` `[BOTH]`

```ts
export const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (opts: ProductsSearch) => [...productKeys.lists(), opts] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: string) => [...productKeys.details(), id] as const,
}
```

Key discipline: **keys are a factory, not string soup.** Invalidation becomes surgical:
`queryClient.invalidateQueries({ queryKey: productKeys.lists() })`.

## 18.4 The loader ↔ query handshake (prefetch + SSR + client) `[BOTH]`

FILE: `src/routes/products/$productId.tsx`

```tsx
export const Route = createFileRoute('/products/$productId')({
  loader: async ({ context: { queryClient }, params }) => {
    // Runs during SSR AND during preload. ensureQueryData = "fetch if absent, reuse if fresh"
    await queryClient.ensureQueryData({
      queryKey: productKeys.detail(params.productId),
      queryFn: () => getProduct({ data: { id: params.productId } }), // server function
    })
  },
  component: ProductPage,
})

function ProductPage() {
  const { productId } = Route.useParams()
  // Suspense-friendly; on first paint the data is ALREADY HERE (dehydrated) — no fetch flash
  const { data: product } = useSuspenseQuery({
    queryKey: productKeys.detail(productId),
    queryFn: () => getProduct({ data: { id: productId } }),
  })
  return <ProductView product={product} />
}
```

The full journey:

```
Route Loader (ensureQueryData) → Query fetch via server fn → SSR render
 → dehydrate(cache) into HTML → browser → hydrate(cache)
 → useSuspenseQuery resolves INSTANTLY from cache → later client nav reuses/refetches by staleTime
```

## 18.5 Mutations, invalidation, polling, dependents

```tsx
// [CLIENT] — mutation + targeted invalidation
const qc = useQueryClient()
const archive = useMutation({
  mutationFn: (orderId: string) => archiveOrder({ data: { orderId } }),
  onSuccess: (_, orderId) => {
    qc.invalidateQueries({ queryKey: orderKeys.lists() })
    qc.invalidateQueries({ queryKey: orderKeys.detail(orderId) })
  },
})

// Live dashboard widget: poll every 15s while mounted
useQuery({ queryKey: ['activity'], queryFn: fetchActivity, refetchInterval: 15_000 })

// Dependent queries: second query waits for first data
const { data: org } = useQuery({ queryKey: ['org', orgId], queryFn: getOrg })
const { data: members } = useQuery({
  queryKey: ['org', orgId, 'members'],
  queryFn: () => listMembers({ data: { orgId: org!.id } }),
  enabled: !!org,                      // ← gate until dependency resolves
})

// Infinite feed (Module 21 pairs this with Virtual)
useInfiniteQuery({
  queryKey: ['feed'],
  queryFn: ({ pageParam }) => getFeed({ data: { cursor: pageParam } }),
  initialPageParam: undefined as string | undefined,
  getNextPageParam: (last) => last.nextCursor,
})
```

Cancellation is automatic on unmount (Query aborts the fetch); server functions ride `fetch` semantics.

## 18.6 Module 07 + Query cache tuning — they compose

- `defaultPreload: 'intent'` → hover runs the loader → `ensureQueryData` warms the Query cache →
  **prefetching works across both layers** with zero extra code.
- Keep `staleTime` policy consistent: if the router revalidates aggressively but Query caches for 5 min,
  you'll see stale data anyway. One policy owner per screen.

## 18.7 Anti-patterns

⚠️ **DO NOT** wrap every loader in Query by reflex — route-scoped data often needs nothing but Module 07.

⚠️ **DO NOT** fetch in a component what the route loader already owns (double fetch, cache split).

⚠️ **DO NOT** use unstable key arrays (`[{ search }]` objects without normalization) — keys must be
deterministic.

⚠️ **DO NOT** `queryClient.clear()` casually on logout and forget to also `router.invalidate()` (or vice
versa) — half-cleared state is a leak.

## 18.8 Exercises

- **Beginner:** Wire 18.2 into your app; add one `useQuery` widget; prove via View Source that its data is in the SSR payload.
- **Intermediate:** Convert `/products` to Query-owned (loader = `ensureQueryData`), with `productKeys` and list invalidation after `createProduct`.
- **Production:** Add a 15s-polled activity widget that pauses when the tab is hidden (`refetchIntervalInBackground: false`), and measure network in DevTools.
- **Architecture Challenge (graded in Module 22):** Revisit Module 12's dashboard challenge — now that you know both caches, finalize your per-feature decision table.
- **Debug Challenge:** After login, user B sees user A's product list for a moment. Which cache(s) survived logout, and what's the correct dual-invalidation fix?

🧠 **MENTAL MODEL — Query:** the router decides **when** data loads; Query decides **how data lives** (keys,
freshness, mutations). dehydrate/hydrate is the bridge that makes SSR and client cache one continuous store.

---

**Next: [Module 19 — TanStack Form: Type-Safe Forms That Talk to Server Functions →](19-forms.md)**
