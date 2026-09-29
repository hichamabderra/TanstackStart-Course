# Module 13 — Middleware: Composition, Ordering, Context

> Middleware is Start's answer to “where does cross-cutting server logic live?” Auth, logging, rate
> limiting, CSP, tracing — composable, ordered, and type-checked. All APIs here are verified against the
> official Middleware guide (current as of 2026-09-28).

---

## 13.1 Two kinds of middleware (this distinction matters)

| | Request middleware | Server Function middleware |
|---|---|---|
| Created with | `createMiddleware()` (default `type: 'request'`) | `createMiddleware({ type: 'function' })` |
| Applies to | **All** server requests: SSR renders, server routes, server functions | Server functions only |
| Methods | `.middleware()`, `.server()` | `.middleware()`, `.validator()`, `.client()`, `.server()` |
| Can depend on | request middleware | request **and** function middleware |
| Key constraint | request middleware **cannot** depend on function middleware (direction!) | |

Function middleware is a *subset* with extras: it can validate input and run logic on the **client** side of
the RPC (before/after the network hop) — think optimistic state or client-side telemetry.

## 13.2 A request middleware: timing + logging `[SERVER]`

FILE: `src/server/middleware/logging.ts`

```ts
import { createMiddleware } from '@tanstack/react-start'

export const loggingMiddleware = createMiddleware().server(async ({ request, next }) => {
  const requestId = crypto.randomUUID()
  const started = performance.now()

  const result = await next()          // ⬅ run the rest of the chain, get its result

  const ms = (performance.now() - started).toFixed(1)
  console.log(JSON.stringify({
    requestId,
    method: request.method,
    url: request.url,
    ms,
    status: result instanceof Response ? result.status : 200,
  }))
  return result                        // ⬅ you may transform/replace the result here
})
```

Rules of `next`:

- **Always call `next()`** unless you're intentionally short-circuiting (returning a `Response` early =
  short-circuit; nothing downstream runs).
- You can inspect `request`, mutate nothing you don't own, and wrap `next()` in try/catch for error policies.

## 13.3 Passing context down the chain: `sendContext` `[SERVER]`

FILE: `src/server/middleware/auth.ts`

```ts
import { createMiddleware } from '@tanstack/react-start'
import { auth } from '~/lib/auth.server'

export const authMiddleware = createMiddleware()
  .middleware([loggingMiddleware])            // composition: depends on logging
  .server(async ({ request, next }) => {
    const session = await auth.api.getSession({ headers: request.headers })
    if (!session) {
      return new Response(JSON.stringify({ error: { status: 401 } }), { status: 401 })
    }
    // Typed context flows to nested middleware AND into server-function handlers
    return next({ sendContext: { session, user: session.user } })
  })
```

Downstream consumption:

```ts
export const whoAmI = createServerFn({ method: 'GET' })
  .middleware([authMiddleware])
  .handler(async ({ context }) => {
    return { id: context.session.user.id }    // typed: came from sendContext
  })
```

> Verify the exact context property names (`sendContext` / `context`) against the Middleware guide when you
> write this — the shape has evolved across Start versions; the *pattern* (outer produces, inner consumes)
> is stable.

## 13.4 Authorization middleware (function middleware) `[SERVER + CLIENT]`

FILE: `src/server/middleware/require-role.ts`

```ts
import { createMiddleware } from '@tanstack/react-start'

export const requireAdmin = createMiddleware({ type: 'function' })
  .middleware([authMiddleware])                     // function mw can depend on request mw
  .server(async ({ context, next }) => {
    if (context.user.role !== 'ADMIN') {
      return new Response(JSON.stringify({ error: { status: 403 } }), { status: 403 })
    }
    return next()
  })
```

Then any function is one line to lock down:

```ts
export const adminListUsers = createServerFn({ method: 'GET' })
  .middleware([requireAdmin])
  .handler(async ({ context }) => usersRepo.list(context.user.orgId))
```

## 13.5 Where middleware attaches — all four seams

1. **Global (every request incl. SSR)** — custom start instance, FILE: `src/start.ts`:

```ts
import { createStart, createCsrfMiddleware } from '@tanstack/react-start'
import { loggingMiddleware } from '~/server/middleware/logging'

const csrfMiddleware = createCsrfMiddleware({
  filter: (ctx) => ctx.handlerType === 'serverFn', // protect server functions (verified pattern)
})

export const startInstance = createStart(() => ({
  requestMiddleware: [csrfMiddleware, loggingMiddleware],
}))
```

> Note: if you have **no** `src/start.ts`, Start installs CSRF protection for server functions automatically.
> Defining `src/start.ts` makes the stack explicit — production apps should define it.

2. **Per server route / per handler** (Module 11): `server.middleware: [...]` and `createHandlers({ GET: { middleware: [...] } })`.
3. **Per server function**: `.middleware([...])` on `createServerFn`.
4. **Route-level UX guards** are NOT middleware: `beforeLoad` (Module 06) is the client-visible counterpart.

## 13.6 Execution order — the onion

```
request
 └▶ global request middleware (order given in requestMiddleware[])
     └▶ route-level server.middleware
         └▶ handler-level middleware
             └▶ function middleware .client() [on client]
                 └▶ function middleware .server()
                     └▶ .validator()
                         └▶ HANDLER
```

Ordering rules that keep you sane:

- **Cheap first** (request-id, logging), **auth before authz**, **authz before rate-limit bookkeeping**? No —
  rate-limit *before* expensive auth DB lookups on anonymous endpoints; after auth when limits are per-user.
  Decide per endpoint; don't cargo-cult.
- Security headers middleware (CSP etc.) should run **early** so even error responses carry headers.

## 13.7 Building the course chain (capstone)

```
logging (requestId, timing)
  → securityHeaders (Module 25)
  → rateLimit (per IP/user, Module 25)
  → csrf (server fns)
  → auth (session → context)
  → authorization (role/permission/ownership)
  → handler (service → repo → DB)
```

## 13.8 Anti-patterns

⚠️ **DO NOT** over-use middleware. If a concern applies to exactly one function, put it in that function.
Middleware is for *cross-cutting* concerns; a chain of 12 bespoke middlewares is unreadable state-passing.

⚠️ **DO NOT** mutate the request/response casually; wrap, don't reach in.

⚠️ **DO NOT** use middleware for business rules (“if org is trial, block checkout”) — that's the service
layer (Module 17). Middleware = transport/policy concerns.

⚠️ **DO NOT** forget: middleware errors are responses too — return your standard error envelope (Module 12),
not bare strings.

## 13.9 Exercises

- **Beginner:** Implement `loggingMiddleware`; hit three endpoints; check ordered logs with request ids.
- **Intermediate:** Implement `authMiddleware` + `requireAdmin` with `sendContext`; prove a USER gets 403 on an admin server function and that the handler sees `context.user`.
- **Production:** Add a `timing` field to every log line (route SSR vs server fn vs server route) and export p50/p95 per endpoint (Module 29 hooks this to real metrics).
- **Architecture Challenge:** You need per-org rate limits and a global CSP. Assign each to the right seam (global mw, route mw, function mw, or service) and explain why the others are wrong.
- **Debug Challenge:** A server function returns 401 intermittently only in production. Using the onion: which layer can 401? How do you bisect? (Hint: add a log at each middleware boundary; suspect session cookie domain/SameSite across hosts — Module 14.)

🧠 **MENTAL MODEL — Middleware:** an onion around every server execution. Request middleware wraps the whole
kitchen; function middleware wraps one dish; `next()` is the doorway; `sendContext` is the plate passed
through it.

---

**Next: [Module 14 — Authentication: Sessions, Cookies, Better Auth →](14-authentication.md)**
