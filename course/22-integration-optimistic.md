# Module 22 — Integration Chapter: Router + Query + Table + Form + Server Functions (+ Optimistic UI)

> Capstone Stages 7–13 converge here. We build the **Admin Orders screen** end-to-end through every layer,
> then teach optimistic mutations properly. This is also where your Module 12 architecture challenge gets
> graded.

---

## 22.1 The full pipeline (one picture to rule them all)

```
URL search params (Module 05)
      ↓  validateSearch: zod schema, defaults
TanStack Router (Module 03/06)
      ↓  loader: ensureQueryData (prefetch/SSR)
TanStack Query (Module 18)
      ↓  queryKey = f(search) → server function via useQuery
Server Function (Module 10)
      ↓  CSRF → middleware auth/authz → validator
Service → Repository → PostgreSQL (Modules 15–17)
      ↓  { rows, total } serialized
TanStack Table renders + reports intent (Module 20) → back to URL
TanStack Form edits a row (Module 19) → mutation server fn → invalidation
```

## 22.2 The Admin Orders screen, piece by piece

FILE: `src/routes/_admin/admin.orders.tsx` `[BOTH]`

```tsx
export const ordersAdminSearchSchema = z.object({
  q: z.string().optional(),
  status: z.enum(['all', 'pending', 'paid', 'shipped', 'cancelled']).default('all'),
  page: z.coerce.number().int().min(1).catch(1),
  pageSize: z.coerce.number().int().oneOf([20, 50, 100]).catch(20),
  sort: z.enum(['createdAt', 'totalCents', 'status']).default('createdAt'),
  dir: z.enum(['asc', 'desc']).default('desc'),
}).default({})

export const Route = createFileRoute('/_admin/admin.orders')({
  validateSearch: (raw) => ordersAdminSearchSchema.parse(raw),
  // Loader warms the QUERY cache — SSR paints the table; preload warms it on hover
  loader: async ({ context: { queryClient }, search }) => {
    await queryClient.ensureQueryData({
      queryKey: adminOrderKeys.list(search),
      queryFn: () => adminListOrders({ data: search }),   // server function
    })
  },
  component: AdminOrdersPage,
  pendingComponent: OrdersTableSkeleton,
})

function AdminOrdersPage() {
  const search = Route.useSearch()
  const { data, isFetching } = useSuspenseQuery({
    queryKey: adminOrderKeys.list(search),
    queryFn: () => adminListOrders({ data: search }),
    placeholderData: (prev) => prev,
  })
  return (
    <section aria-labelledby="orders-heading">
      <header className="flex items-center justify-between">
        <h1 id="orders-heading">Orders</h1>
        <OrdersToolbar />              {/* search/status/page-size → URL (replace:true) */}
      </header>
      <OrdersAdminTable rows={data.rows} total={data.total} fetching={isFetching} />
    </section>
  )
}
```

FILE: `src/server/orders.functions.ts` `[SERVER]` (the server side of the same screen)

```ts
export const adminListOrders = createServerFn({ method: 'GET' })
  .validator(ordersAdminSearchSchema)            // SAME schema as the URL — one source of truth
  .middleware([requirePermission('orders:manage')])
  .handler(async ({ data, context }) => {
    return ordersRepo.listForOrg(context.user.orgId, data) // scoped SQL, LIMIT/OFFSET, index hit
  })

export const updateOrderStatus = createServerFn({ method: 'POST' })
  .validator(z.object({ orderId: z.uuid(), status: OrderStatus }))
  .middleware([requirePermission('orders:manage')])
  .handler(async ({ data, context }) => ordersService.setStatus(data, context.user))
```

FILE: `src/features/orders/edit-order-dialog.tsx` `[CLIENT]` (Form in a dialog)

```tsx
export function EditOrderDialog({ order }: { order: AdminOrderRow }) {
  const qc = useQueryClient()
  const form = useForm({
    defaultValues: { status: order.status },
    validators: { onSubmit: z.object({ status: OrderStatus }) },
    onSubmit: async ({ value }) => {
      await updateOrderStatus({ data: { orderId: order.id, status: value.status } })
      qc.invalidateQueries({ queryKey: adminOrderKeys.lists() })
      qc.invalidateQueries({ queryKey: adminOrderKeys.detail(order.id) })
    },
  })
  // …shadcn Dialog + form fields (Module 19/23), focus trap & Esc close included
}
```

Every arrow in 22.1's diagram is one of these files. Same pattern scales to Users and Products screens.

## 22.3 Optimistic UI — when and how

**When it's right:** low-risk, high-frequency, easily rolled-back actions — favorite, archive, toggle,
status flip. **When it's wrong:** money movement, destructive deletes, anything with side effects you can't
undo (send the real mutation first there).

The anatomy:

```
UI update (instant)  →  server mutation  →  success: confirm + invalidate
                                          →  failure: ROLLBACK + error toast
```

FILE: `src/features/products/favorite-button.tsx` `[CLIENT]`

```tsx
export function FavoriteButton({ product }: { product: ProductSummary }) {
  const qc = useQueryClient()
  const key = productKeys.detail(product.id)

  const toggle = useMutation({
    mutationFn: () => toggleFavorite({ data: { productId: product.id } }),

    onMutate: async () => {
      await qc.cancelQueries({ queryKey: key })         // stop refetches clobbering optimism
      const previous = qc.getQueryData<ProductDetail>(key)
      if (previous) {
        qc.setQueryData(key, { ...previous, isFavorite: !previous.isFavorite }) // optimistic write
      }
      return { previous }                                // snapshot for rollback
    },

    onError: (_err, _vars, ctx) => {
      if (ctx?.previous) qc.setQueryData(key, ctx.previous)  // ← rollback
      toast.error("Couldn't update favorite — reverted.")
    },

    onSettled: () => qc.invalidateQueries({ queryKey: key }), // truth wins eventually
  })

  return (
    <button
      aria-pressed={/* from query data */ undefined}
      onClick={() => toggle.mutate()}
    >
      ♥
    </button>
  )
}
```

The three-capstone examples, mapped:

| Action | Optimistic? | Rollback surface | Notes |
|---|---|---|---|
| Favorite product | ✅ | `isFavorite` flag | classic case above |
| Update profile | ✅ (fields) | previous profile values | Form keeps values; toast on revert |
| Archive order | ⚠️ cautious | row status | server may reject (permissions/invoices) — optimistic status badge + confirm |
| Toggle notification | ✅ | boolean | tiny, frequent, zero-risk |

Rules:

1. **Always roll back** on error — an optimistic lie that persists is worse than a spinner.
2. Optimistic writes go through the **same cache keys** the reads use (or `router` cache via
   `router.invalidate()` for loader-owned data).
3. Keep a subtle “pending” affordance (dimmed icon) — instant ≠ pretending nothing happened.
4. Server still re-validates everything; optimism is presentation, not trust.

## 22.4 Grading your Module 12 architecture challenge

The senior answer for the authenticated org dashboard:

| Feature | Data loading | State | Server boundary | Cache |
|---|---|---|---|---|
| Tenant users list | Route loader → `ensureQueryData` | URL (page/sort/q) | `adminListUsers` SF + `requirePermission('users:manage')` | Query key by search; invalidate on role change |
| Analytics (slow) | **Streamed** deferred promise (Module 09) | none | `getAnalytics` SF | short `staleTime` or none; never cache across tenants without tenant in key |
| Live notifications | Client `useQuery` polling/SSE | Query | `getNotifications` SF | Query; 15s refetch; **no** shared caching |
| Searchable orders | URL search → loader → Query | URL | `adminListOrders` SF (org-scoped) | Query by search; invalidate on status change |
| Admin permissions | beforeLoad context (session/role) | context | every SF re-checks | — |

**Cached:** catalog-like lists keyed by search (per-user/per-org by construction of the request).
**Never cached:** personalized dashboards across users, anything without tenant identity in the key,
mutation responses.
**Authorization:** enforced in middleware (coarse) + service (resource) + SQL (`orgId`) — Module 15 — never
only in `beforeLoad`.

## 22.5 Anti-patterns

⚠️ **DO NOT** mix ownership of one widget between loader cache and Query (decide per feature, Module 18).

⚠️ **DO NOT** optimistic-update money or irreversible actions.

⚠️ **DO NOT** invalidate `['everything']` after every mutation — you built keys for a reason.

⚠️ **DO NOT** skip `cancelQueries` in `onMutate` — a racing refetch silently reverts your optimistic write.

## 22.6 Exercises

- **Beginner:** Implement favorite-with-optimism exactly as 22.3; force a server error (temporarily) and watch rollback.
- **Intermediate:** Ship the full Admin Orders screen: toolbar, table, edit dialog, all URL-synced.
- **Production:** Add optimistic order-status updates with permission-aware UI + audit log entry server-side; measure perceived latency with/without optimism.
- **Architecture Challenge:** Notifications must feel live. Compare polling (Query), SSE via a server route, and WebSocket — pick one for Meridian and defend it against ops cost, proxy compatibility, and battery.
- **Debug Challenge:** “Favorite heart flips, then flips back a second later.” Using the mutation lifecycle, name the three suspects (missing cancelQueries, refetchInterval clobbering, invalidate ordering) and fix each.

🧠 **MENTAL MODEL — Integration:** URL → Router → Query → Server → DB is one pipeline; each library owns one
station. Optimistic UI is a *prediction at the station you own*, reconciled against the train when it arrives.

---

**Next: [Module 23 — UI System: Tailwind v4, shadcn/ui, Motion, Accessibility →](23-ui-styling-animation-a11y.md)**
