# Module 16 — PostgreSQL + Drizzle: Schema, Migrations, Transactions

> Capstone Stage 4. The database is the bottom of every data flow in this course. We evaluate the ORM
> decision honestly, pick one primary tool, and enforce one rule: **the database is server-only, full stop.**

---

## 16.1 Prisma vs Drizzle (2026 evaluation)

| Criterion | Drizzle ORM (`0.45.3`) | Prisma (`7.x`) |
|---|---|---|
| Mental model | SQL-first: query builder mirrors SQL; you see the query | Schema-first DSL + generated client |
| Codegen step | None (types come from your table definitions) | Generate client (Prisma 7 ships a lighter, driver-adapter-based client) |
| Edge/Workers | Excellent (tiny runtime, `postgres.js`/`neon`/D1 drivers) | Good on Workers via driver adapters (the official `start-basic-auth` example uses Prisma 7 + `@prisma/adapter-libsql`) |
| Raw SQL escape hatch | First-class `sql` template | `$queryRaw` |
| Migrations | `drizzle-kit` (SQL migrations you can read/edit) | Prisma Migrate (declarative) |
| Ecosystem/aid | Younger, very active | Mature tooling, Studio, huge track record |
| Best for | Teams that want SQL control + minimal runtime | Teams that want batteries + polish |

**Course decision: Drizzle as primary** — SQL visibility is a teaching advantage, the runtime is edge-friendly
for the Cloudflare deployment module, and there's no codegen in CI. **Prisma is the documented alternative**;
everything we teach (repositories, scoping, transactions) maps 1:1. Whichever you pick, the *architecture*
(Modules 17) matters more than the query syntax.

## 16.2 Connection — server-only by construction `[SERVER]`

FILE: `src/server/db/index.server.ts`

```ts
import '@tanstack/react-start/server-only'   // file marker: importing this from client code fails the build
import { drizzle } from 'drizzle-orm/node-postgres'
import pg from 'pg'
import * as schema from './schema'

// One pool per server process — NOT per request, NOT per function call
const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,  // server-only env (Module 29)
  max: 10,                                     // keep modest; multiple server instances multiply this
})

export const db = drizzle(pool, { schema })
```

> On **Cloudflare Workers** you'd swap to a Workers-compatible driver (e.g. Neon's HTTP driver or
> `postgres.js` over Hyperdrive) — same Drizzle surface (Module 30). The lesson: *drivers differ by runtime;
> the repository layer shouldn't notice.*

## 16.3 Schema — Meridian core entities `[SERVER]`

FILE: `src/server/db/schema.ts`

```ts
import { pgTable, uuid, text, integer, boolean, timestamp, index, uniqueIndex, pgEnum } from 'drizzle-orm/pg-core'

export const roleEnum = pgEnum('role', ['USER', 'ADMIN'])

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
  role: roleEnum('role').notNull().default('USER'),
  passwordHash: text('password_hash').notNull(),           // never serialized to clients
  emailVerified: boolean('email_verified').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
})

export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
})

export const memberships = pgTable('memberships', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  orgRole: text('org_role', { enum: ['OWNER', 'MEMBER'] }).notNull().default('MEMBER'),
}, (t) => [uniqueIndex('memberships_user_org_uq').on(t.userId, t.orgId)])

export const products = pgTable('products', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }), // 🔒 tenant scope
  name: text('name').notNull(),
  description: text('description'),
  priceCents: integer('price_cents').notNull(),
  stock: integer('stock').notNull().default(0),
  archived: boolean('archived').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('products_org_idx').on(t.orgId),
  index('products_org_name_idx').on(t.orgId, t.name),       // search-within-tenant
])

export const orders = pgTable('orders', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organizations.id),   // 🔒 tenant scope
  userId: uuid('user_id').notNull().references(() => users.id),
  status: text('status', { enum: ['pending', 'paid', 'shipped', 'cancelled'] }).notNull().default('pending'),
  totalCents: integer('total_cents').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('orders_org_created_idx').on(t.orgId, t.createdAt),
  index('orders_user_idx').on(t.userId),
])

export const orderItems = pgTable('order_items', {
  id: uuid('id').primaryKey().defaultRandom(),
  orderId: uuid('order_id').notNull().references(() => orders.id, { onDelete: 'cascade' }),
  productId: uuid('product_id').notNull().references(() => products.id),
  quantity: integer('quantity').notNull(),
  unitPriceCents: integer('unit_price_cents').notNull(),     // snapshot price — never join at read time
})
```

Schema lessons embedded above:

- **`orgId` on tenant-owned tables** → tenant isolation lives in the data model (Module 26).
- **Composite indexes match real queries** (`orgId + createdAt` for the orders dashboard).
- **Money is integer cents** — floating point has no place near money.
- **Snapshots over joins** for historical facts (order item price at purchase time).
- Better Auth owns its own tables (`sessions`, `accounts`, …) via the Drizzle adapter — don't hand-edit them.

## 16.4 Queries — builder, relations, raw SQL `[SERVER]`

```ts
import { and, desc, eq, ilike, sql } from 'drizzle-orm'

// Filtered, paginated list (Module 05 feeds `opts`)
export async function listProducts(orgId: string, opts: ProductListOpts) {
  const where = and(
    eq(products.orgId, orgId),                 // tenant scope — non-negotiable
    eq(products.archived, false),
    opts.q ? ilike(products.name, `%${opts.q}%`) : undefined,
  )
  const rows = await db.select().from(products).where(where)
    .orderBy(opts.sort === 'price-asc' ? products.priceCents : desc(products.createdAt))
    .limit(opts.pageSize)
    .offset((opts.page - 1) * opts.pageSize)

  const [{ count: total }] = await db.select({ count: sql<number>`count(*)` })
    .from(products).where(where)

  return { rows, total }
}

// Relations — avoid N+1 by asking for the join:
const order = await db.query.orders.findFirst({
  where: eq(orders.id, orderId),
  with: { items: { with: { product: true } } },   // one composed query, not N
})
```

## 16.5 Transactions — correctness boundaries `[SERVER]`

```ts
export async function placeOrder(input: PlaceOrderInput, user: SessionUser) {
  return db.transaction(async (tx) => {
    const order = await tx.insert(orders).values({
      orgId: user.orgId, userId: user.id, totalCents: input.totalCents,
    }).returning()

    await tx.insert(orderItems).values(input.items.map((i) => ({
      orderId: order[0]!.id, productId: i.productId,
      quantity: i.quantity, unitPriceCents: i.unitPriceCents,
    })))

    // stock decrement with guard: UPDATE … WHERE stock >= qty
    const updated = await tx.update(products)
      .set({ stock: sql`${products.stock} - ${i.quantity}` })
      .where(and(eq(products.id, i.productId), sql`${products.stock} >= ${i.quantity}`))
    if (updated.rowCount === 0) throw new ConflictError('insufficient-stock')

    return order[0]
  }) // any throw above rolls everything back
}
```

Rules: keep transactions short; no HTTP calls inside them; idempotency keys outside-in (Module 12).

## 16.6 Migrations with drizzle-kit

```bash
npm i -D drizzle-kit
# drizzle.config.ts points at ./src/server/db/schema.ts + DATABASE_URL
npx drizzle-kit generate   # emits SQL migration files → COMMIT THEM
npx drizzle-kit migrate    # applies (local/staging)
```

Production migration discipline → Module 30 (zero-downtime expand/contract, never destructive in one step).

## 16.7 Performance checklist (expand in Module 27)

1. `EXPLAIN ANALYZE` every hot query; aim for index scans, not seq scans.
2. Index every `WHERE orgId = …` combo you actually query.
3. Kill N+1 with `with:` relations or `inArray` batching.
4. `LIMIT/OFFSET` for tables under ~100k rows; keyset pagination beyond (Module 20 sidebar).
5. Connection pool sized to instances × max ≤ provider limit.

## 16.8 Anti-patterns

⚠️ **DO NOT** import `db` from anything a component can reach (no loaders-without-server-functions, no
`useEffect` DB calls). Import protection enforces this — respect the red squiggles.

⚠️ **DO NOT** create one client per request (connection exhaustion). One pool per process.

⚠️ **DO NOT** store money as floats or compute totals client-side (recompute server-side, always).

## 16.9 Exercises

- **Beginner:** Model the schema above, run `generate` + `migrate` against local Postgres (Docker), and insert seed data with a script.
- **Intermediate:** Implement `listProducts` with filtering + both sort modes; verify index usage with `EXPLAIN`.
- **Production:** Implement `placeOrder` transaction with the stock guard; write a concurrency test (two parallel orders, one unit of stock → exactly one succeeds).
- **Architecture Challenge:** When is `LIMIT/OFFSET` wrong, and how does keyset pagination (`WHERE (createdAt, id) < (:cursor)`) change your URL schema from Module 05?
- **Debug Challenge:** Dashboard slows from 40ms to 2s as orders grow. `EXPLAIN` shows a seq scan on `orders`. Which index is missing given the query shape in 16.3, and why didn't the PK index help?

🧠 **MENTAL MODEL — Database:** the schema encodes the security model (`orgId` everywhere), the indexes
encode the read patterns, transactions encode the invariants — and the whole thing lives behind a
server-only door.

---

**Next: [Module 17 — Server Architecture: Services, Repositories & Validation Everywhere →](17-service-layer-validation.md)**
