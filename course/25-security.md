# Module 25 — Security: The Full Production Threat Model

> Capstone-quality security, not checkbox security. Structure: threat → mechanism in Start → code. The
> guiding law: **the browser is hostile territory; the server is the only adult in the room.**

---

## 25.1 Who is responsible for what

| Layer | Owns | Examples |
|---|---|---|
| **Frontend** | UX of security | hiding unavailable actions, CSRF-friendly same-origin calls, never rendering raw HTML |
| **Server (Start)** | **All real decisions** | authn/authz, validation, secrets, headers, rate limits |
| **Database** | Last-line invariants | constraints, FKs, `orgId` NOT NULL, row-level checks |
| **Infrastructure** | Transport & isolation | TLS, secrets storage, network policy, patching |

If a control exists in only one layer, it must be the server (or below).

## 25.2 XSS — injection into rendering `[CLIENT + SERVER]`

React escapes strings by default, which kills 95% of XSS. The remaining 5%:

```tsx
// ❌ DO NOT — user/DB content becomes code
<div dangerouslySetInnerHTML={{ __html: product.description }} />

// ✅ sanitize server-side at WRITE time (store sanitized), render as HTML only via a vetted sanitizer
// ✅ or render markdown with a strict allowlist renderer
```

Rules: sanitize at the boundary where content is **stored**; CSP (25.6) is the safety net; report rendering
of any HTML in code review as a security event.

## 25.3 CSRF — forgery of your user's session `[SERVER]`

Start ships `createCsrfMiddleware()` and **auto-applies it to server functions** unless you define
`src/start.ts` (verified). Then add it explicitly:

```ts
// src/start.ts [SERVER]
import { createStart, createCsrfMiddleware } from '@tanstack/react-start'

export const startInstance = createStart(() => ({
  requestMiddleware: [
    createCsrfMiddleware({ filter: (ctx) => ctx.handlerType === 'serverFn' }),
    loggingMiddleware,
  ],
}))
```

Defense-in-depth: session cookies are `SameSite=Lax` (Module 14); server functions are same-origin-only by
design; server routes that mutate for browsers re-check origin; external APIs use signed requests instead
(webhooks, Module 11).

## 25.4 IDOR & authorization bypass — the #1 SaaS breach `[SERVER]`

Insecure Direct Object Reference: `GET /api/orders/123` where `123` belongs to another tenant. Counter is
the Module 15 stack: **fetch → policy check with the resource → scoped SQL.** Plus:

- Never trust ids from the client for *which tenant* — the tenant comes from the **session**, always.
- Cross-tenant ids answer **404** (existence hiding) per policy (15.6).
- Add regression tests: *for every resource endpoint, a cross-tenant request must fail* — generated as a
  suite from your route list.

## 25.5 SSRF — the server fetching where it's told `[SERVER]`

Any server function that fetches a **user-supplied URL** (webhooks, avatar-from-URL, imports) is an SSRF
candidate:

```ts
// ❌ DO NOT
const res = await fetch(data.url)   // data.url could be http://169.254.169.254/ (cloud metadata!) or internal hosts

// ✅ allowlist schemes (https only), resolve DNS, block private/link-local ranges, set timeouts, cap size
const safe = await assertSafeUrl(data.url)   // host allowlist + SSRF checks
const res = await fetch(safe, { signal: AbortSignal.timeout(5000), redirect: 'manual' })
```

## 25.6 Security headers via middleware `[SERVER]`

FILE: `src/server/middleware/security-headers.ts`

```ts
export const securityHeaders = createMiddleware().server(async ({ next }) => {
  const result = await next()
  if (!(result instanceof Response)) return result
  const headers = new Headers(result.headers)
  headers.set('content-security-policy',
    "default-src 'self'; script-src 'self'; img-src 'self' data: https://cdn.example.com; " +
    "object-src 'none'; base-uri 'self'; frame-ancestors 'none'")
  headers.set('x-content-type-options', 'nosniff')
  headers.set('referrer-policy', 'strict-origin-when-cross-origin')
  headers.set('permissions-policy', 'camera=(), microphone=(), geolocation=()')
  headers.set('strict-transport-security', 'max-age=63072000; includeSubDomains')
  return new Response(result.body, { status: result.status, statusText: result.statusText, headers })
})
```

Notes: start CSP strict (no `'unsafe-inline'`); Vite's dev mode needs relaxations only in dev. Streaming SSR
sets headers **before** the first flush — register this middleware early (Module 13 ordering).

## 25.7 SQL injection — nearly free, but not entirely `[SERVER]`

Drizzle parameterizes everything you build with its API. The risk surfaces where you write raw SQL:

```ts
// ❌ string concatenation in sql``
sql`SELECT * FROM products WHERE name LIKE '%${q}%'`
// ✅ Drizzle's parameter interpolation
sql`SELECT * FROM products WHERE name LIKE ${'%' + q + '%'}`
// ✅ or better: the query builder (Module 16)
```

Same discipline for `orderBy`: accept only allowlisted column names, never interpolate sort fields from the URL.

## 25.8 CORS `[SERVER]`

Server functions are same-origin (no CORS surface). For public server routes: default = **no CORS headers**
(browsers block cross-origin reads). Add explicit, narrow `Access-Control-Allow-Origin` only for routes that
need it (public API), never `*` with credentials.

## 25.9 Rate limiting `[SERVER]`

FILE: `src/server/middleware/rate-limit.ts` (sketch)

```ts
const buckets = new Map<string, { count: number; reset: number }>() // in-memory per instance;
// production: Redis/Workers KV shared store (Module 30)

export function rateLimit({ windowMs, max, key }: { windowMs: number; max: number; key: (req: Request) => string }) {
  return createMiddleware().server(async ({ request, next }) => {
    const id = key(request)
    const now = Date.now()
    const b = buckets.get(id)
    if (!b || now > b.reset) buckets.set(id, { count: 1, reset: now + windowMs })
    else if (++b.count > max) throw RateLimited(Math.ceil((b.reset - now) / 1000))
    return next()
  })
}

// Usage: rateLimit({ windowMs: 60_000, max: 5, key: (r) => 'login:' + ipOf(r) }) on auth endpoints
```

Rate-limit login, registration, password reset, and any expensive endpoint — **before** expensive auth DB
lookups where possible.

## 25.10 Secrets management `[SERVER]`

1. Server-only env vars (never `VITE_*`) for credentials (Module 02/29).
2. `.server.ts` files + import protection keep secret-reading modules out of client bundles.
3. Production secrets live in the host's secret store (Workers secrets, Railway env, vaults) — **never**
   in git, logs, or error payloads.
4. Rotate on suspicion; Better Auth `secret` rotation plan included in runbooks.

## 25.11 Supply chain & logging

- Lockfiles committed; `npm audit` + Dependabot in CI (Module 30); review new deps like hiring decisions.
- **Redact** logs: cookies, `authorization`, tokens, emails in some jurisdictions. Structured logging with
  explicit fields (Module 29) makes redaction a formatter, not a hope.

## 25.12 Threat-model cheat sheet (tape to monitor)

| Threat | Primary control | Secondary |
|---|---|---|
| XSS | escape/sanitize rendering | CSP |
| CSRF | CSRF middleware + SameSite | same-origin server fns |
| IDOR | policy + scoped SQL (Module 15) | 404 existence hiding |
| Session theft | HttpOnly/Secure cookies | short expiry, revocation |
| SQLi | parameterized ORM | allowlisted sorts |
| SSRF | URL allowlist + checks | timeouts, redirect=manual |
| Credential stuffing | rate limit + lockout | MFA (Module 14) |
| Tenant leak | `orgId` in every query | cross-tenant test suite |
| Secret leak | server-only env + files | log redaction |

## 25.13 Exercises

- **Beginner:** Add the security-headers middleware; score the app with a header-checker; fix CSP until strict passes in dev AND prod.
- **Intermediate:** Build the cross-tenant regression suite (25.4) for products/orders/users.
- **Production:** Shared rate limiter (Redis) keyed by user-or-IP with 429 + `Retry-After`; load-test login lockout.
- **Architecture Challenge:** A feature needs “import product data from URL”. Write the security spec before any code (scheme/host allowlists, size caps, content-type checks, storage policy).
- **Debug Challenge:** Pen test reports: “server function callable cross-site via a `<form>` POST from evil.com”. Given Start's CSRF model, why did this happen (custom `src/start.ts` without CSRF middleware) and what's the fix + regression test?

🧠 **MENTAL MODEL — Security:** every request is hostile input until proven otherwise; policy lives next to
the resource; headers, cookies, and limits are the moat; the database is the keep. Defense in depth means
each layer assumes the previous one failed.

---

**Next: [Module 26 — Multi-Tenancy, File Uploads & Background Jobs →](26-multi-tenancy-uploads-jobs.md)**
