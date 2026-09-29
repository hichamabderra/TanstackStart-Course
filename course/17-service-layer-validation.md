# Module 17 — Server Architecture: Services, Repositories & Validation Everywhere

> Capstone architecture day. Two skills separate hobby apps from production ones: a **layered server
> boundary** and **runtime validation at every trust crossing**. This module builds both.

---

## 17.1 The layered server — and why each layer exists

```
Server Function / Server Route        ← RPC/HTTP surface: parse, call, shape response
        ↓
Authorization                         ← policy module (Module 15): WHO may do WHAT to THIS
        ↓
Service                               ← business rules & orchestration (transactions, events)
        ↓
Repository                            ← data access: queries, scoping, pagination (Drizzle)
        ↓
Database                              ← Postgres
```

FILE: `src/server/orders.service.ts` `[SERVER]`

```ts
import { ordersRepo, productsRepo } from './repos.server'
import { assertCanViewOrder } from './authz/guards'
import { placeOrderTx } from './db/transactions.server'

export const ordersService = {
  async create(input: ValidatedOrderInput, user: SessionUser) {
    // 1. business rules (prices recomputed server-side — never trust client totals)
    const priced = await priceOrder(input.items)
    // 2. invariants in a transaction
    return placeOrderTx({ ...input, totalCents: priced.totalCents }, user)
  },

  async get(orderId: string, user: SessionUser) {
    const order = await ordersRepo.findByIdWithItems(orderId)
    assertCanViewOrder(user, order)          // ← authz next to the resource, not in middleware alone
    return order
  },
}
```

FILE: `src/server/orders.functions.ts` `[SERVER]` — the surface stays THIN

```ts
export const createOrder = createServerFn({ method: 'POST' })
  .validator(OrderInputSchema)
  .handler(async ({ data }) => {
    const session = await ensureSession()
    return ordersService.create(data, session.user)   // no SQL here. ever.
  })
```

### Why not just write Drizzle calls in handlers?

- **Duplicated queries** (“everyone” writes the orders join slightly differently).
- **Forgotten scoping** (one handler omits `orgId` → tenant leak).
- **Untestable business rules** (can't unit-test a rule fused to RPC plumbing).

### And why not more layers?

⚠️ **DO NOT** build Domain-Driven cathedrals on day one: no “UnitOfWorkFactoryManager”. Add a service
method when a *second* caller or a *second* rule appears. Premature abstraction is its own anti-pattern;
the course rule is **thin surface, growing middle**.

## 17.2 Validation — TypeScript is compile-time; Zod is runtime

TypeScript evaporates at runtime and at every network boundary. **Every crossing gets a schema:**

```
Browser form ──(1)──▶ Server Function .validator ──(2)──▶ Service (typed by inference)
Server Route request ──▶ (3) boundary parse        DB rows ──(4)──▶ external API responses
process.env ──(5)──▶ server startup                URL search ──(6)──▶ validateSearch (Module 05)
```

FILE: `src/server/schemas.ts` `[BOTH]` — one home for shared schemas

```ts
import { z } from 'zod'

export const Money = z.number().int().min(0).max(100_000_000)

export const ProductInputSchema = z.object({
  name: z.string().trim().min(1).max(120),
  description: z.string().max(5000).optional(),
  priceCents: Money,
  categoryId: z.uuid(),
})
export type ProductInput = z.infer<typeof ProductInputSchema>   // types COME from schemas

export const OrderInputSchema = z.object({
  items: z.array(z.object({ productId: z.uuid(), quantity: z.number().int().min(1).max(99) }))
    .min(1).max(50),
})
```

### (5) Environment validation — fail at boot, not at 3am

FILE: `src/server/env.server.ts` `[SERVER]`

```ts
import '@tanstack/react-start/server-only'
import { z } from 'zod'

const Env = z.object({
  DATABASE_URL: z.string().min(1),
  BETTER_AUTH_SECRET: z.string().min(32),
  BETTER_AUTH_URL: z.url(),
  STRIPE_SECRET_KEY: z.string().optional(),
})

export const env = Env.parse(process.env)   // crash immediately on misconfiguration
```

### Parsing strategy at boundaries

```ts
// Trusted-ish internal paths: throw (fail loud)
const input = ProductInputSchema.parse(body)

// Public HTTP: safeParse → structured 422
const parsed = ProductInputSchema.safeParse(body)
if (!parsed.success) {
  return jsonError(422, { issues: parsed.error.issues })  // Zod 4: error.issues is canonical
}
```

Rules:

1. **One schema per concept** reused by form (Module 19), server function `.validator`, and server-route
   boundary. Duplicate schemas drift; drift becomes exploits.
2. Validate **at the boundary**, then trust the typed value inside (don't re-validate per layer).
3. Schemas are `[BOTH]` — they're the rare module that safely ships to client and server.

## 17.3 Full data flow, both directions (capstone reference)

### GET PRODUCTS

```
Browser → TanStack Router → Route (/products, validateSearch)
 → Loader (loaderDeps = search)
 → listProducts server function  [SERVER]
    → ensureSession + hasPermission('products:read')
    → productsService.list(opts, user)
    → productsRepo.listForOrg(orgId, opts)   (scoped SQL)
    → Postgres (index scan)
 → { rows, total } serialized                [SERVER → CLIENT]
 → router cache (Module 07) / Query (Module 18)
 → UI renders (SSR first, then client)
```

### CREATE PRODUCT

```
Browser form (TanStack Form, Module 19)
 → client-side schema check (instant UX)
 → createProduct server function             [SERVER]
    → CSRF check → authMiddleware → requirePermission('products:create')
    → .validator(ProductInputSchema)          (runtime truth)
    → productsService.create(input, user)     (recompute/sanitize)
    → productsRepo.insertForOrg(...)
    → return Product
 → client: invalidate(['products']) / router.invalidate()
 → list updates (optimistic variant: Module 22)
```

## 17.4 Anti-patterns

⚠️ **DO NOT** validate only on the client (trivially bypassed) or only on the server (terrible UX). Both,
same schema.

⚠️ **DO NOT** write `as` casts at boundaries instead of parsing — a cast is a lie you tell the compiler.

⚠️ **DO NOT** let services import from `createServerFn` modules (inverted dependency); services are pure
server code that functions call.

⚠️ **DO NOT** return DB rows with `passwordHash` to any client — map to DTOs at the function boundary
(have the repository return already-safe shapes).

## 17.5 Exercises

- **Beginner:** Split your products feature into `functions.ts` / `service.ts` / `repo.server.ts` per 17.1.
- **Intermediate:** Add `env.server.ts` validation; boot the app with a missing `DATABASE_URL` and observe the clean startup failure.
- **Production:** Introduce DTO mapping so no repository type ever reaches a client bundle; enforce with a type-level test.
- **Architecture Challenge:** Where would you put “recompute order totals” and why is `clientTotal` from the form a *suggestion*, never a fact?
- **Debug Challenge:** A form submits `{ priceCents: "1999" }` (string). Client schema says error, server says error, but a bug report says an order went through at $0. Reconstruct the likely layering failure.

🧠 **MENTAL MODEL — Layers:** functions are doors, services are rules, repositories are plumbing, schemas are
locks on every door — and types are what you get *after* the lock picks fail.

---

**Next: [Module 18 — TanStack Query in Start (and the router-cache decision) →](18-tanstack-query.md)**
