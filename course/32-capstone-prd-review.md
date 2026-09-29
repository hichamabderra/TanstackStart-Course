# Module 32 — Capstone: PRD, Your Design First, Senior Architecture Review & the Debugging Gym

> The finish line. First the full requirements document for **Meridian**. Then — deliberately — *no
> implementation*. You design; the checklist reviews. Finally, the Debugging Gym: broken code you must fix.

---

## 32.1 PRD — “Meridian”: Multi-Tenant SaaS Operations & Commerce Platform

### Business requirements

Organizations run their product catalog, orders, and internal operations from one place. Meridian provides a
public storefront per org's catalog, self-serve signup, role-based internal tools, and admin analytics —
with strict tenant isolation as the non-negotiable requirement.

### Users & roles

| Role | Scope | Capabilities |
|---|---|---|
| `USER` | own data + own org | profile, browse, favorites, cart, orders, notifications |
| `ORG_OWNER` | their organization | members, org settings, tenant resources |
| `ORG_MEMBER` | their organization | org-scoped daily work (per permission set) |
| `ADMIN` (platform) | cross-org | manage users/products/orders, analytics, permissions |

Permissions are data (`products:manage`, `orders:read:org`, `users:manage`, `analytics:view`,
`org:settings:update`, …) bundled by role (Module 15).

### Pages (route tree, condensed)

```
PUBLIC    /  /pricing  /products  /products/$productId
AUTH      /login  /register  /verify-email  /reset-password
APP       /dashboard  /profile  /settings  /orders  /orders/$orderId
          /notifications  /favorites
ORG       /org/members  /org/roles  /org/settings
ADMIN     /admin/users  /admin/products  /admin/orders  /admin/analytics
```

### Entities

`users · sessions(auth) · organizations · memberships · products · categories · orders · order_items ·
favorites · notifications · uploads · audit_events · idempotency_keys`

### API surface

**Server Functions (internal RPC):** session helpers, product/order CRUD, favorites, cart, notifications
read/mark, org member management, admin lists, dashboard summary, analytics, uploads, exports (streaming).

**Server Routes (HTTP):** `/api/auth/$` (Better Auth), `/api/webhooks/stripe`, `/api/v1/products` (public
catalog, cached), `/rss.xml`, health check, file download via signed URLs.

### Cross-cutting decisions (course answers baked in)

| Concern | Decision |
|---|---|
| Caching | Router cache for route-owned reads; Query for shared/live data; public catalog: CDN-friendly headers; dashboards: never shared-cached |
| Authentication | Better Auth (DB sessions, HttpOnly cookies, email OTP verification, 2FA, rate-limited login) |
| Authorization | Policies module + per-endpoint middleware + org-scoped SQL; cross-tenant 404 |
| Security | CSRF middleware, security headers, IDOR suite, redacted logs, secrets validated at boot |
| Accessibility | WCAG AA, four-state surfaces, keyboard-complete flows, reduced-motion |
| Performance | Budgets (Module 27.1), intent preloading, streaming dashboards, CI bundle gate |
| Testing | Pyramid 70/20/10 + cross-tenant suite + axe in CI |
| Deployment | Primary: Cloudflare Workers (+Neon); fallback: Docker/Node. Migrations expand/contract |
| Observability | requestId everywhere, per-route timing, error classes, job metrics |

## 32.2 YOUR assignment (design before looking at any code)

Using only the PRD and your course knowledge, produce:

1. **Route tree** — full file layout incl. pathless layouts, server routes, lazy splits.
2. **Folder structure** — features vs server layers; where `.server.ts` boundaries sit.
3. **Database schema** — tables, FKs, indexes; mark every tenant-scoped column.
4. **Server/client boundary map** — table of every capability → server fn / server route / client-only.
5. **Cache strategy** — per screen: router vs Query, staleTime, invalidation triggers.
6. **Authentication flow** — signup→verify→login→session→revocation.
7. **Authorization matrix** — role × action × enforcement layer.
8. **API/server function architecture** — function list w/ validators + middleware chains.
9. **Deployment architecture** — host, runtime choices, CI/CD, migration policy.

Then review your design against 32.3. (In a mentored setting, this is where a senior reviews it with you —
the checklist *is* the review.)

## 32.3 Senior-engineer architecture review checklist

**Route structure** — tree mirrors information architecture? layouts reused via pathless routes? lazy
splits on heavy screens? server routes namespaced (`/api/...`)?

**Server/client boundaries** — any DB/secret access reachable from a component? loaders free of secrets?
`*.server.ts` used for every infra module? import protection enabled?

**Middleware** — chain ordered (cheap→auth→authz)? no business rules in middleware? CSRF registered if
`src/start.ts` exists?

**Authentication** — cookies flagged correctly? sessions revocable? login rate-limited? caches cleared on
login/logout?

**Authorization** — every server fn/route authorizes itself? resource policies take the resource? tenant
scope in every list query signature? cross-tenant test suite present?

**Data loading** — loaders coordinate, server functions compute? parallel where independent? streaming for
slow widgets?

**Caching** — cache keys tenant-safe? invalidation after every mutation that matters? no personalized HTML
in shared caches?

**Query usage** — not reflexive; shared/live/mutation-heavy data only? one ownership model per widget?

**Database** — indexes match hot queries? transactions around multi-write operations? money as cents?

**Validation** — one schema per concept reused form→validator→boundary? env validated at boot?

**Security** — Module 25 table satisfied? headers middleware early? uploads sanitized? SSRF-checked fetches?

**Testing** — pyramid balanced? authz + money paths integration-heavy? E2E journeys green in CI?

**Deployment** — build reproducible? migrations automated + non-destructive? rollback runbook exists?

**Observability** — requestId end-to-end? per-route timing? 5xx alerts + 4xx dashboards? secrets redacted?

**Verdict format (use it on yourself):** ✅ good decisions / ⚠️ risks / ❌ must-fix — with one-line reasons.

## 32.4 THE DEBUGGING GYM — 15 broken scenarios

For each: **broken code → repro → diagnose → fix → mental model.** (Solutions are sketched; reproduce them
yourself first.)

### 1. Incorrect route path after rename
**Broken:** file renamed to `products.$slug.tsx`; links still `params={{ productId }}`.
**Repro:** `tsc` fails (that's the feature). **Diagnose:** compiler lists every consumer. **Fix:** update to
`slug`. **Mental model:** route ids are schema; renames are migrations.

### 2. Search param type mismatch
**Broken:** `search={{ page: '2' }}` — schema expects `number`. **Repro:** compile error (good) or runtime
NaN if schema coerces elsewhere. **Fix:** pass `2`; schemas coerce *URL* strings, not your typed calls.
**Mental model:** URL arrives as strings; typed APIs don't.

### 3. Stale loader cache after mutation
**Broken:** favorite toggles but list unchanged until hard refresh. **Diagnose:** no invalidation; route
match still fresh (`staleTime`). **Fix:** `router.invalidate()` (or lower `staleTime` / move to Query).
**Mental model:** caches don't observe your mutations — you tell them.

### 4. Bad loader dependencies
**Broken:** loader reads `search.page` but `loaderDeps` omits it → pagination reuses page-1 data.
**Diagnose:** deps comparison says “unchanged”. **Fix:** include everything the loader consumes in
`loaderDeps`. **Mental model:** deps ARE the cache key inputs.

### 5. Incorrect preloading
**Broken:** expensive `/admin` pages preload on hover → wasted auth checks + rate-limit noise. **Fix:**
`preload={false}` on privileged links; preload intent-only on cheap, likely targets. **Mental model:**
preloading is eager execution, not free magic.

### 6. Broken auth redirect
**Broken:** `_authed` redirects to `/login` without `redirect` param → user lands on `/` after login.
**Fix:** `throw redirect({ to: '/login', search: { redirect: location.href } })` and honor it post-login.
**Mental model:** auth UX must remember intent.

### 7. Missing authorization (the classic)
**Broken:** `getOrder` fetches by id only — any logged-in user reads any order. **Repro:** curl with
another tenant's id. **Fix:** fetch → `assertCanViewOrder` → scoped repo (Module 15). **Mental model:**
authentication ≠ authorization; check the resource.

### 8. Insecure server function
**Broken:** `deleteUser` trusts `data.userId`, no permission check. **Fix:** require session; require
`users:manage`; never accept “act as someone else” params from the client. **Mental model:** every function
is a public door with your name on it.

### 9. Tenant-leaking query
**Broken:** `listProducts(opts)` makes `orgId` optional “for admin”. **Repro:** org B sees org A rows.
**Fix:** mandatory `orgId` in signature; admin cross-org lists are separate, permissioned, audited.
**Mental model:** optional scoping is a future breach.

### 10. N+1 query
**Broken:** orders table renders `<UserName id={row.userId}/>` fetching per row → 100 requests. **Diagnose:**
query-count middleware (Module 29). **Fix:** relation `with: { user }` in the list query. **Mental model:**
list pages issue O(1) queries or they're wrong.

### 11. Hydration mismatch
**Broken:** `new Date().toLocaleTimeString()` in first render. **Diagnose:** console warning + flicker.
**Fix:** server picks value → ships it; client renders the shipped value. **Mental model:** first client
render must equal server render deterministically.

### 12. Wrong server/client boundary
**Broken:** `import fs from 'node:fs'` in a module a component imports. **Diagnose:** build error via import
protection (or worse: a secret leaks when protection is off). **Fix:** move to `*.server.ts` behind a
server function. **Mental model:** boundaries are enforced by tooling, not discipline.

### 13. Incorrect QueryClient lifecycle
**Broken:** `new QueryClient()` inside a component → cache resets per render; SSR payload never hydrates.
**Diagnose:** constant refetching; empty initial state. **Fix:** one client in `getRouter()` (Module 18.2).
**Mental model:** the client is app state; construct once.

### 14. Stale Query cache after logout
**Broken:** logout navigates away; next user sees previous user's cached dashboard. **Fix:**
`queryClient.clear()` + `router.invalidate()` on session change (both directions: login too). **Mental
model:** caches are per-user; session change = cache change.

### 15. Optimistic rollback failure
**Broken:** `onMutate` updates cache but `onError` forgets `setQueryData(ctx.previous)` → heart stays
favorited after 403. Also: missing `cancelQueries` lets a refetch resurrect the write. **Fix:** full
Module 22.3 lifecycle. **Mental model:** optimism is a prediction; errors demand reconciliation.

## 32.5 Graduation

You can now narrate the whole journey — **request → router → middleware → loaders → server → data →
render → stream → hydrate → navigate** — and you know where every lock on the door lives. That's the actual
outcome this course promised: not “I know TanStack Start”, but **“I can design, secure, test, and ship a
production system on it.”**

Keep three habits: verify against the docs map (Module 31.4) before trusting any snippet (including this
course, someday); measure before optimizing; and review your own designs with 32.3 before anyone else does.

**Build Meridian. Then build the next thing.**
