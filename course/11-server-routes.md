# Module 11 — Server Routes: HTTP Endpoints for the Outside World

> Server Functions are for *your* app. **Server Routes** are for everyone else: webhooks, public APIs, RSS
> feeds, OAuth callbacks, anything that speaks plain HTTP across an origin or to a third party.

---

## 11.1 The current API — server routes live inside the route tree

```ts
// 🔁 OLD: createServerFileRoute('/api/hello', { methods: { GET: ... } })  — deprecated style
// ✅ MODERN: a `server` block on a file route (verified in the official Server Routes guide)
```

FILE: `src/routes/api/health.ts` `[SERVER]`

```ts
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/api/health')({
  server: {
    handlers: {
      GET: async ({ request }) => {
        return new Response(JSON.stringify({ ok: true }), {
          headers: { 'content-type': 'application/json' },
        })
      },
    },
  },
})
```

Key facts (all verified):

- Server routes live **alongside UI routes** in `src/routes/` and follow the same file conventions
  (`$param`, `$.ts` catch-all, escaped dots).
- A single file can serve **both** a UI route (`component`) and HTTP handlers (`server`) — same path, two
  behaviors by HTTP method/accept.
- Requests are matched by Start's handler (`createStartHandler` in a custom `src/server.ts` — Module 30).
- **One handler file per path** — duplicates error at build time.

## 11.2 Full CRUD endpoint with validation and auth `[SERVER]`

FILE: `src/routes/api/products.ts`

```ts
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'
import { ProductInput } from '~/server/schemas'
import { productsRepo } from '~/server/products.repo.server'

export const Route = createFileRoute('/api/products')({
  server: {
    middleware: [apiAuthMiddleware], // route-level: applies to ALL methods (Module 13)
    handlers: {
      GET: async ({ request }) => {
        const url = new URL(request.url)
        const page = z.coerce.number().int().min(1).parse(url.searchParams.get('page') ?? 1)
        const { rows, total } = await productsRepo.listPublic({ page })
        return json({ items: rows, total, page }, { headers: cacheControlPublic(60) })
      },
      POST: async ({ request }) => {
        const body = await request.json().catch(() => null)
        const parsed = ProductInput.safeParse(body)
        if (!parsed.success) return jsonError(422, parsed.error)   // validation error
        const session = await requireApiSession(request)           // 401 if missing
        if (!can(session.user, 'products:create')) return jsonError(403)
        const product = await productsRepo.create(parsed.data, session.user.orgId)
        return json(product, { status: 201 })
      },
      PUT: async ({ request }) => { /* full replace → same guards */ },
      PATCH: async ({ request }) => { /* partial update via PartialInput schema */ },
      DELETE: async ({ request }) => { /* ownership check → 204 */ },
    },
  },
})

// [SERVER] — src/server/http.ts — tiny helpers, consistent responses
export function json(data: unknown, init?: ResponseInit) {
  return new Response(JSON.stringify(data), {
    ...init, headers: { 'content-type': 'application/json', ...init?.headers },
  })
}
export function jsonError(status: number, error?: unknown) {
  return json({ error: { status, message: messageFor(status, error) } }, { status })
}
```

## 11.3 Per-handler middleware with `createHandlers` `[SERVER]`

```ts
export const Route = createFileRoute('/api/orders')({
  server: {
    handlers: ({ createHandlers }) =>
      createHandlers({
        GET: {
          middleware: [rateLimitMiddleware, loggingMiddleware],
          handler: async ({ request }) => json(await ordersRepo.listForRequest(request)),
        },
        POST: {
          middleware: [csrfApiMiddleware, auditMiddleware],
          handler: async ({ request }) => { /* … */ },
        },
      }),
  },
})
```

Ordering: **route-level middleware runs first, then handler-level** (verified). Both compose with the same
`createMiddleware` machinery as server functions (Module 13).

## 11.4 Path conventions recap (server-route flavored)

| File | Endpoint |
|---|---|
| `api/users.ts` | `/api/users` |
| `api/users/$id.ts` | `/api/users/:id` |
| `api/auth.$.ts` | `/api/auth/*` — **catch-all** (how Better Auth is mounted, Module 14) |
| `rss.xml.ts` (escaped: `rss[.]xml.ts`) | `/rss.xml` |

## 11.5 Webhook example — the canonical server-route use case `[SERVER]`

FILE: `src/routes/api/webhooks/stripe.ts`

```ts
import { createFileRoute } from '@tanstack/react-router'
import { stripe } from '~/server/stripe.server'

export const Route = createFileRoute('/api/webhooks/stripe')({
  server: {
    handlers: {
      POST: async ({ request }) => {
        const signature = request.headers.get('stripe-signature')
        const raw = await request.text()                    // ⚠️ raw body for signature check
        let event
        try {
          event = stripe.webhooks.constructEvent(raw, signature, process.env.STRIPE_WEBHOOK_SECRET!)
        } catch {
          return new Response('Invalid signature', { status: 400 }) // never 500 for bad signatures
        }
        switch (event.type) {
          case 'checkout.session.completed':
            await ordersService.markPaid(event.data.object) // idempotent! (see 12.x)
            break
        }
        return new Response('ok')
      },
    },
  },
})
```

Webhook rules: **verify signatures**, read the **raw body** before parsing, be **idempotent** (deliveries
retry), respond fast (enqueue heavy work — Module 26), log everything.

## 11.6 Cookies, headers, uploads

```ts
import { setCookie, getCookie, deleteCookie } from '@tanstack/react-start/server'

// Set an HttpOnly session cookie from a server route:
setCookie('session', token, {
  httpOnly: true, secure: true, sameSite: 'Lax', path: '/', maxAge: 60 * 60 * 24 * 7,
})

// Multipart upload (full treatment in Module 26):
POST: async ({ request }) => {
  const form = await request.formData()
  const file = form.get('avatar')
  if (!(file instanceof File)) return jsonError(422)
  // validate size/type → stream to object storage
}
```

## 11.7 Anti-patterns

⚠️ **DO NOT** build your own app's internal mutations as server routes “for flexibility” — you lose RPC
typing and take on manual fetch plumbing (Module 12 matrix).

⚠️ **DO NOT** skip auth inside handlers because “it's under `/api`” — paths are not a security boundary.

⚠️ **DO NOT** return `200` for errors with an error body; use correct status codes (Module 12/24).

## 11.8 Exercises

- **Beginner:** Build `/api/health` and `/api/echo` (POST returns the JSON body verbatim). Test with `curl`.
- **Intermediate:** Implement the full `/api/products` CRUD with Zod validation and the four correct status codes (201/204/422/403).
- **Production:** Add a signed webhook endpoint with replay protection (event-id dedupe table); write an E2E test that replays the same event twice.
- **Architecture Challenge:** Which of these are server routes vs server functions: `/rss.xml`, “mark notification read”, Stripe webhook, “change password”, public partner API, SSE feed? Justify each with Module 12's matrix (next).
- **Debug Challenge:** A webhook works in dev but 400s in production. List the three usual suspects (body parsed by middleware before signature check, wrong secret, proxy altering bytes) and how you'd verify each.

🧠 **MENTAL MODEL — Server Routes:** the part of your Start app that is “just an HTTP API” — same tree, same
middleware, same type-safety on the inside; raw Request/Response on the outside.

---

**Next: [Module 12 — Server Functions vs Server Routes · BFF & API Architecture →](12-server-fn-vs-routes-api.md)**
