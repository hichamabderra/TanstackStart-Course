# Module 10 — Server Functions In Depth

> Server Functions are Start's flagship: **type-safe, same-origin RPC from anywhere in your app to the
> server.** Everything you'd put in an API route for *your own UI* becomes a server function instead.

---

## 10.1 WHAT / WHY / WHERE / WHEN / HOW

**WHAT.** A function defined with `createServerFn()` whose *handler* runs only on the server but can be
called from components, loaders, hooks, or other server functions — with inputs and outputs serialized and
type-checked across the boundary.

**WHY.** Your UI needs server capabilities (DB, secrets, file system) without exposing them. Options:
hand-written REST endpoints (drift-prone, manual typing), GraphQL (another layer), or **RPC that the compiler
checks end-to-end**. Server functions are the third option, integrated with middleware, auth, caching, and
the request lifecycle.

**WHERE.** Anywhere in app code. The build replaces handler implementations with RPC stubs in the client
bundle — *server code never ships to the browser* (verified: “Static Imports Are Safe”; avoid dynamic
imports).

**WHEN.** For all internal application mutations and reads that your own UI triggers. Not for third-party
consumers — that's a Server Route (Module 11).

**HOW.**

FILE: `src/server/products.functions.ts` `[SERVER handler, callable from BOTH]`

```ts
import { createServerFn } from '@tanstack/react-start'
import { z } from 'zod'
import { productsRepo } from './products.repo.server'   // .server.ts → import-protected

const ProductInput = z.object({
  name: z.string().min(1).max(120),
  priceCents: z.number().int().min(0),
  categoryId: z.string().uuid(),
})

export const createProduct = createServerFn({ method: 'POST' })
  .validator(ProductInput)                    // 🔁 MODERN name (was .inputValidator)
  .handler(async ({ data }) => {
    // [SERVER] — full Node/server API surface here
    const session = await requireSession()    // Module 14; throws 401 if none
    requirePermission(session.user, 'products:create') // Module 15
    return productsRepo.create({ ...data, orgId: session.user.orgId })
  })
```

Calling it from the client:

```tsx
// [CLIENT]
import { createProduct } from '~/server/products.functions' // ✅ static import is safe

function NewProductForm() {
  const fn = useServerFn(createProduct) // hook form → stable identity + error handling
  return (
    <form
      onSubmit={(e) => {
        e.preventDefault()
        fn({ data: { name, priceCents, categoryId } })
          .then((product) => navigate({ to: '/products/$productId', params: { productId: product.id } }))
      }}
    >
      …
    </form>
  )
}
```

## 10.2 The mental wire format

```
CLIENT                                        SERVER
fn({ data }) ──▶ HTTP (same origin) ──▶  CSRF check → function middleware → validator
                                            → handler ({ data, request, context })
                                            → return value (must be serializable)
◀── JSON payload ───────────────────────── (or a Response, incl. streams — Module 09)
```

- `method: 'GET'` (default) → cacheable semantics, used for reads. `method: 'POST'` → mutations; POST bodies
  also accept `FormData` (uploads, Module 26).
- **Serialization is type-checked** by default (“strict” mode): inputs and returns must be serializable.
  Opt out deliberately with `createServerFn({ strict: false })` — almost never needed.
- Server functions are **same-origin by design**; Start verifies `Sec-Fetch-Site`/`Origin`/`Referer` via
  built-in CSRF middleware (auto-installed unless you define `src/start.ts`, then you add
  `createCsrfMiddleware()` explicitly — Module 25).

## 10.3 Where to call them (verified patterns)

```tsx
// 1) Route loader [BOTH]
export const Route = createFileRoute('/products')({
  loader: () => listProducts({ data: { page: 1 } }),
})

// 2) Component via hook [CLIENT]
function List() {
  const getProducts = useServerFn(listProducts)
  const { data } = useQuery({ queryKey: ['products'], queryFn: () => getProducts() }) // Module 18
}

// 3) From another server function [SERVER] — composition
export const archiveOrder = createServerFn({ method: 'POST' })
  .validator(z.object({ orderId: z.string() }))
  .handler(async ({ data }) => {
    const session = await requireSession()
    await assertCanManageOrder(session.user, data.orderId)
    return ordersService.archive(data.orderId, session.user.orgId) // → invalidation, events
  })
```

## 10.4 Accessing the request inside a server function `[SERVER]`

```ts
import { getRequestHeaders } from '@tanstack/react-start/server'

export const getSession = createServerFn({ method: 'GET' }).handler(async () => {
  const headers = getRequestHeaders()            // the current request's headers (cookies live here)
  return auth.api.getSession({ headers })
})
```

This is the official pattern (used by Better Auth's own Start guide): session resolution flows through the
incoming request's headers.

## 10.5 Errors across the boundary

- **Expected errors** (validation, 401, 403, 404, conflicts): throw typed errors from the handler; catch on
  the caller. Model them explicitly:

```ts
// [SERVER] — src/server/errors.ts
export class AuthError extends Error { status = 401 }
export class ForbiddenError extends Error { status = 403 }
export class NotFoundError extends Error { status = 404 }
export class ConflictError extends Error { status = 409 }
```

  Start surfaces handler errors to the caller as rejected promises (module 24 standardizes the mapping,
  including any `HttpError` helper your Start version ships — check the *Error Handling* section of the
  server-functions guide).
- **Unexpected errors**: let them throw → logged server-side (Module 29), generic message to client
  (never leak stack traces).
- **Validation errors**: `.validator` rejections arrive as structured Zod errors — TanStack Form (Module 19)
  maps them onto fields.

## 10.6 File organization (official recommendation)

```
src/server/
├── products.functions.ts   # createServerFn wrappers — safe to import from anywhere
├── products.service.ts     # business rules (server code, but not RPC surface)
├── products.repo.server.ts # Drizzle queries; .server.* = import-protected from client
└── schemas.ts              # Zod schemas — client-safe, shared
```

Rules: `.functions.ts` = the RPC surface (thin); `*.server.ts` = anything touching DB/secrets (protected);
plain `.ts` = shared types/schemas. Modules 16/17 build this for real.

## 10.7 Progressive enhancement & forms

Server functions accept `FormData` on POST, so plain `<form action>`-style flows can degrade gracefully, and
Module 19 shows TanStack Form submitting JSON with typed field errors. Either way: **validate with Zod in the
validator — the client is never trusted.**

## 10.8 Anti-patterns

⚠️ **DO NOT** `await import()` server functions dynamically — bundler can't stub correctly (verified warning).

⚠️ **DO NOT** return non-serializable things (Date objects are fine only as strings; no classes, no streams
except via `Response`).

⚠️ **DO NOT** treat “it's only called from my UI” as a security model — the endpoint is HTTP; validate +
authorize inside the handler (Modules 14/15/25).

⚠️ **DO NOT** build public APIs with server functions (no CORS story, CSRF intentionally blocks cross-origin)
— use server routes (Module 11).

## 10.9 Exercises

- **Beginner:** Port the counter example: `getCount` (GET) + `addToCount` (POST with `.validator((d: number) => d)`), called from a button with `router.invalidate()`.
- **Intermediate:** Implement `listProducts` with the Module 05 search schema as its validator; call it from the route loader AND from a `useQuery` in a component.
- **Production:** Add a request-id middleware (Module 13) and log every server-function call with duration + status; assert no secret ever appears in client-visible errors.
- **Architecture Challenge:** Where do you draw the line between “many small server functions” vs “few fat ones”? Decide for: product CRUD, favorites, cart, checkout. Consider cache keys, authorization granularity, and RPC round-trips.
- **Debug Challenge:** A server function works in dev but fails in production with a CORS-like rejection. Using the CSRF model above, diagnose the cause (hint: a deployment proxy stripped the `Origin` header) and choose the least-dangerous fix.

🧠 **MENTAL MODEL — Server Functions:** an internal, typed RPC surface. The function signature IS the API
contract; the build system enforces the boundary; middleware enforces the policy; serialization enforces the
wire.

---

**Next: [Module 11 — Server Routes: HTTP Endpoints for the Outside World →](11-server-routes.md)**
