# Module 12 — Server Functions vs Server Routes · BFF & API Architecture

> The most common production mistake in Start apps is using the wrong boundary. This module gives you the
> decision matrix, then zooms out to where Start sits in your overall system architecture.

---

## 12.1 The decision matrix

| Question | Server Function | Server Route |
|---|---|---|
| Who calls it? | **Your** Start app (client or server) | Anyone: third parties, curl, webhooks, other services |
| Protocol | Same-origin RPC (Start-owned) | Plain HTTP (you own semantics) |
| Type safety | End-to-end (input/output checked) | Whatever you build (Zod at the boundary) |
| Serialization | Automatic | Manual (`Response`) |
| CSRF protection | Built-in (same-origin checks) | Your responsibility (signatures, tokens) |
| CORS / cross-origin | Not the tool | Yes — you configure it |
| Caching semantics | Via router/Query layers | Standard HTTP caching headers |
| Progressive enhancement | Yes (FormData capable) | Yes (it's HTTP) |
| Ideal for | Mutations + reads for your own UI | Webhooks, public APIs, feeds, OAuth callbacks, integrations |

### Worked decisions (memorize the shape of these)

| Requirement | Choice | Why |
|---|---|---|
| `createOrder()` from checkout UI | **Server Function** | Internal, typed, needs session + org context |
| Stripe webhook (`/api/webhooks/stripe`) | **Server Route** | External caller, signature verification, raw body |
| Public partner API (`/api/v1/products`) | **Server Route** | Third-party clients, versioning, API keys, CORS |
| “Toggle favorite” button | **Server Function** | Internal mutation, optimistic UI (Module 22) |
| `/rss.xml`, `/sitemap.xml` | **Server Route** | Crawlers/consumers outside the app |
| Better Auth handler mount | **Server Route** (`/api/auth/$`) | Library expects an HTTP handler |
| CSV export triggered by your UI | **Server Function** returning `Response` | Internal trigger; needs custom headers/stream |

**Heuristic:** if you can imagine someone who isn't your frontend calling it, it's a server route.

## 12.2 TanStack Start as a Backend-for-Frontend (BFF)

```
        Browser / mobile app
              │
              ▼
   ┌─────────────────────────┐
   │      TanStack Start     │  ← BFF: auth, shaping, aggregation,
   │  server fns + routes    │     frontend-specific endpoints
   └────┬─────────┬──────────┘
        │         │
   Internal    External
   services    services (payments, email, search)
        │
        ▼
     PostgreSQL
```

The BFF pattern: Start is the *only* backend your UI knows. It aggregates internal services, enforces
auth, and shapes responses for the UI. Three configurations, honestly compared:

| Architecture | When it's right | When it hurts |
|---|---|---|
| **Start-only backend** (server fns + Drizzle directly) | Greenfield SaaS, small–mid team, one frontend. **Our capstone default.** | If 5+ clients need the same API |
| **Start as BFF** over existing services | Existing microservices/internal APIs; UI-specific aggregation; hiding legacy | Maintaining two backends; latency hop |
| **Separate backend** (Start = frontend only) | Huge orgs, API platform teams, multiple frontends, strict team boundaries | Type-safety across the wire requires extra tooling (OpenAPI codegen etc.) |

> **WHY TANSTACK?** Server functions make the BFF nearly free: each function *is* a frontend-shaped endpoint,
> typed to the caller. In Next.js-style codebases this layer becomes untyped API-route spaghetti.

## 12.3 API architecture for your server routes

When you DO expose HTTP, do it properly.

### Status codes (the ones you'll actually use)

| Code | Meaning | Meridian example |
|---|---|---|
| `200` | OK | GET product |
| `201` | Created | POST product |
| `204` | No Content (success, no body) | DELETE favorite |
| `400` | Malformed request | Bad JSON, bad signature |
| `401` | **Not authenticated** | No/invalid session |
| `403` | Authenticated but **forbidden** | USER hitting admin endpoint |
| `404` | Not found (or hidden existence) | Order not visible to caller |
| `409` | Conflict | Duplicate slug, version clash |
| `422` | Validation failed | Zod error payload |
| `429` | Rate limited | Retry-After header |
| `500` | Unexpected | Generic message, full log server-side |

### Consistent error envelope

```json
{
  "error": {
    "status": 422,
    "message": "Validation failed",
    "code": "VALIDATION_ERROR",
    "issues": [{ "path": ["priceCents"], "message": "Expected number, received string" }]
  }
}
```

One envelope for every server route; Module 24 reuses its shape in server functions and the UI.

### Pagination / filtering / sorting conventions

```
GET /api/v1/products?page=2&pageSize=20&sort=-createdAt&category=laptops&q=pro
→ 200 { items: [...], page: 2, pageSize: 20, total: 183 }
```

Rules: `sort=-field` for descending; envelope includes `total` (or cursors for huge tables); reject bad
values with `422`, never silently clamp; document.

### Idempotency (for POST endpoints that create things)

```ts
// Client sends: Idempotency-Key: 9d2f… ; server stores (key → response) for 24h
const existing = await idempotencyStore.get(key)
if (existing) return existing.response
// else execute, store under key (unique constraint = race protection), respond
```

Webhooks retry; mobile networks double-submit; checkout buttons get double-clicked. Idempotency keys are
cheap insurance.

## 12.4 Anti-patterns

⚠️ **DO NOT** create a server route “because it feels more standard” when a server function suffices — you
pay manual fetch, manual types, manual error mapping for nothing.

⚠️ **DO NOT** create server functions for things external systems must call — CSRF protection will fight you
and the RPC envelope isn't a public contract.

⚠️ **DO NOT** version public APIs by hope (`/api/products` today, `/api/products-v2` someday). Start with
`/api/v1/...`.

## 12.5 Exercises

- **Beginner:** Take your Module 11 CRUD and add the full status-code matrix + error envelope. `curl` every branch.
- **Intermediate:** Add idempotency-key support to `POST /api/v1/orders`; write a test that submits twice and asserts one order.
- **Production:** Draft the public API contract for Meridian's catalog (fields, pagination, auth via API key, rate limits) as an OpenAPI-style spec document — no implementation yet.
- **Architecture Challenge (from the course brief):** You have an authenticated org dashboard with tenant-specific users, analytics, live notifications, searchable orders, and admin permissions. Decide per feature: route loader? TanStack Query? URL search params? Server Function? Server Route? What is cached, what is never cached, where is authorization enforced? *Write your answer before Module 22's integration chapter — we'll grade it there.*
- **Debug Challenge:** A mobile team complains server-function-based endpoints “can't be called from our app”. Explain precisely why (same-origin RPC, CSRF, RPC envelope) and propose the correct architecture (BFF with server routes or a versioned API layer).

🧠 **MENTAL MODEL — Boundaries:** server functions = *typed internal organs*; server routes = *skin*. Both
live in one body (the route tree), but only the skin touches the outside world.

---

**Next: [Module 13 — Middleware: Request + Function, Composition, Ordering →](13-middleware.md)**
