# Module 15 — Authorization & RBAC: Roles, Permissions, Ownership

> Capstone Stage 6. If authentication answers **“WHO ARE YOU?”**, authorization answers **“WHAT ARE YOU
> ALLOWED TO DO?”**. Most breaches in real apps are authorization bugs — so this module is ruthless about
> *where* authorization must live.

---

## 15.1 The two-question discipline

| | Authentication | Authorization |
|---|---|---|
| Question | Who are you? | What may you do? |
| Proven by | Session (Module 14) | Policies evaluated **per action, per resource** |
| Fails with | `401` | `403` (or `404` to hide existence) |
| Frequency | Once per request | Once per *operation* (a request can do several) |

**Frontend authorization is UX. Server authorization is security.** Hiding the admin button doesn't stop
`curl`. Both exist; only one of them is load-bearing.

## 15.2 Meridian's role model

```
PLATFORM ROLES:              USER, ADMIN
ORG ROLES (Module 26):       ORG_OWNER, ORG_MEMBER  (per organization membership)
PERMISSIONS (fine grain):    products:read, products:create, products:update, products:delete,
                             orders:read:own, orders:read:org, orders:manage,
                             users:manage, analytics:view, org:settings:update …
OWNERSHIP:                   resources carry orgId (+ optional createdBy) — scoping at the query boundary
```

Design rules:

1. **Roles are bundles of permissions**, never `if (user.role === 'admin')` sprinkled in code.
2. **Ownership checks take the resource**, not just the user: `canViewOrder(user, order)`.
3. Permissions are **data**, so the UI can render capabilities from the same source the server enforces.

## 15.3 The policy layer — one home for all rules `[SERVER]`

FILE: `src/server/authz/policies.ts`

```ts
import type { SessionUser } from '~/types/auth'
import type { Order, Product } from '~/types/entities'

export const rolePermissions: Record<Role, readonly Permission[]> = {
  USER:  ['products:read', 'orders:create', 'orders:read:own', 'profile:update'],
  ADMIN: [/* everything USER has, plus: */
          'users:manage', 'products:manage', 'orders:manage', 'analytics:view'],
} satisfies Record<Role, Permission[]>

export function hasPermission(user: SessionUser, permission: Permission): boolean {
  return rolePermissions[user.role]?.includes(permission) ?? false
}

// Resource-aware policies — the real workhorses:
export function canViewOrder(user: SessionUser, order: Order): boolean {
  if (hasPermission(user, 'orders:manage')) return true          // admin: any order
  return order.orgId === user.orgId && order.userId === user.id  // user: own order only
}

export function canEditProduct(user: SessionUser, product: Product): boolean {
  return hasPermission(user, 'products:manage') && product.orgId === user.orgId
}

export function canManageUsers(user: SessionUser): boolean {
  return hasPermission(user, 'users:manage')
}
```

FILE: `src/server/authz/guards.ts`

```ts
export class ForbiddenError extends Error { readonly status = 403 }

export function assertCan(user: SessionUser, permission: Permission) {
  if (!hasPermission(user, permission)) throw new ForbiddenError()
}

export function assertCanViewOrder(user: SessionUser, order: Order | null) {
  if (!order || !canViewOrder(user, order)) throw new ForbiddenError()
  // (throwing 403 vs 404 for missing is a product decision — see 15.6)
}
```

> ⚠️ **DO NOT** scatter `if (user.role …)` through loaders, components, and handlers. One policy module;
> everything imports it. When security review asks “where do we decide who can delete products?”, the answer
> must be one file.

## 15.4 Enforcement at every layer (defense in depth)

```mermaid
flowchart LR
  A[Request] --> B[Middleware: session → user]
  B --> C{Permission middleware<br/>requirePermission('products:manage')}
  C -->|fail| F[403]
  C -->|pass| D[Server Function handler]
  D --> E[Service: assertCanViewOrder(user, order)]
  E --> G[Repository: WHERE orgId = :orgId<br/>— tenant scope baked into SQL]
```

### Layer 1 — Middleware (coarse, per endpoint) `[SERVER]`

```ts
export const adminListUsers = createServerFn({ method: 'GET' })
  .middleware([requireAdmin])          // Module 13
  .handler(async ({ context }) => usersRepo.list(context.user.orgId))
```

### Layer 2 — Service (resource-aware) `[SERVER]`

```ts
export const ordersService = {
  async getOrder(orderId: string, user: SessionUser) {
    const order = await ordersRepo.findById(orderId)      // unscoped fetch…
    assertCanViewOrder(user, order)                        // …then policy decides
    return order
  },
}
```

### Layer 3 — Repository/SQL (tenant scoping by construction) `[SERVER]`

```ts
// List endpoints NEVER accept an unscoped query:
async listForOrg(orgId: string, opts: OrderQuery) {
  return db.select().from(orders)
    .where(and(eq(orders.orgId, orgId), …opts.filters))
}
```

The `orgId` predicate isn't a filter option — it's mandatory in the function signature. Module 26 makes this
a compiler-enforced property of the repository layer.

### Layer 4 — UI (UX only) `[CLIENT]`

```tsx
// Render capabilities from the session — but NEVER as the only check:
const { user } = Route.useRouteContext()
{hasPermission(user, 'products:create') && <NewProductButton />}
```

Plus a route guard for whole areas:

FILE: `src/routes/_admin.tsx` `[BOTH]`

```tsx
export const Route = createFileRoute('/_admin')({
  beforeLoad: async ({ context }) => {
    const session = await getSession()
    if (!session) throw redirect({ to: '/login' })
    if (session.user.role !== 'ADMIN') throw redirect({ to: '/dashboard' }) // UX
    return { adminUser: session.user }
  },
})
```

## 15.5 Worked flow — “USER tries to archive someone else's order”

```
Browser: archiveOrder({ orderId: 'orgB-order-9' })
  → CSRF ok (same origin) ✔
  → authMiddleware: session valid → user = Alice (USER, orgA) ✔ (authenticated!)
  → handler: ordersService.archive('orgB-order-9', alice)
      → repo.findById → order.orgId = orgB
      → assertCanViewOrder(alice, order) → ✘ ForbiddenError
  → 403 { error: { status: 403, code: 'FORBIDDEN' } }
```

Authentication passed; **authorization failed** — exactly the 401 vs 403 distinction, in motion.

## 15.6 Product-security details senior engineers argue about

- **403 vs 404:** returning `404` for forbidden resources hides their *existence* (good for cross-tenant
  privacy); returning `403` is honest but leaks “this id exists”. Default: `404` across tenant boundaries,
  `403` within them. Decide per resource; document it.
- **Check after fetch, always.** `DELETE /orders/:id` must load the order, check ownership, then delete —
  not “delete where id=X and user=Y” and count rows (fine for simple cases, but error semantics suffer).
- **Capability payloads:** expose `permissions: string[]` on the session for UI; re-verify server-side per
  action anyway.
- **Audit:** every authz *failure* is a security event → structured log with user, action, resource (Module 29).

## 15.7 Anti-patterns

⚠️ **DO NOT** authorize only in `beforeLoad`/middleware and skip the resource check — middleware can't see
the `order` your URL points at.

⚠️ **DO NOT** ship permission decisions to the client and trust them (“the client said `allowed: true`”).

⚠️ **DO NOT** let repositories accept optional `orgId` — optional scoping is a future breach.

⚠️ **DO NOT** encode permissions only in role strings inside conditionals; make them data (15.3).

## 15.8 Exercises

- **Beginner:** Implement 15.3's policy module + `requireAdmin` middleware; write unit tests for every permission mapping.
- **Intermediate:** Add `_admin` route guard + `403` error page; verify a USER navigating to `/admin/users` is redirected and a direct server-function call gets `403`.
- **Production:** Make `orgId` scoping mandatory in repository signatures; add a lint/type test proving no list query compiles without it; add audit logging for all authz failures.
- **Architecture Challenge:** When do you need *resource-level ACLs* (per-document sharing) instead of RBAC? Design the upgrade path for Meridian's products (share a draft product with one external reviewer).
- **Debug Challenge:** “Admins can edit any product, but we just found org A's admin edited org B's product.” Find the bug in 15.4's layers and design the regression test.

🧠 **MENTAL MODEL — Authorization:** identity is global, permission is *per action on a specific resource*,
and tenant scope is *part of the query*. Three checks, three layers, one policy file.

---

**Next: [Module 16 — PostgreSQL + Drizzle: Schema, Migrations, Transactions →](16-database.md)**
