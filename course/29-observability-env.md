# Module 29 — Observability & Environment Variables

> Capstone Stage 18. You can't fix what you can't see, and you can't rotate what you can't find. Two
> unglamorous modules' worth of production survival, in one.

---

## 29.1 Observability = logs + metrics + traces (+ errors)

| Signal | Answers | Meridian implementation |
|---|---|---|
| **Structured logs** | what happened, request by request | JSON logs via request middleware |
| **Request IDs** | correlate one request across logs/errors/support | `x-request-id` in + out |
| **Error tracking** | what broke, how often, in which release | Sentry (or similar) with requestId + user/org context |
| **Metrics** | rates, p50/p95, saturation | endpoint timing, DB timing, queue depth |
| **Traces** | where the time went across layers | server timing per stage (middleware → loader → DB) |

## 29.2 Structured logging middleware `[SERVER]`

FILE: `src/server/middleware/observability.ts`

```ts
export const observability = createMiddleware().server(async ({ request, next }) => {
  const requestId = request.headers.get('x-request-id') ?? crypto.randomUUID()
  const t0 = performance.now()
  let status = 200

  try {
    const result = await next()
    if (result instanceof Response) status = result.status
    return result
  } catch (err) {
    status = 500
    logger.error({ requestId, err: sanitize(err), url: request.url }) // full detail server-side
    throw err
  } finally {
    logger.info({
      requestId,
      method: request.method,
      path: new URL(request.url).pathname,
      status,
      ms: +(performance.now() - t0).toFixed(1),
    })
  }
})
```

Rules:

- **JSON only in production** — fields over prose; grep is a poor man's observability.
- **Redact** `cookie`, `authorization`, tokens, emails where required (Module 25.11).
- Log **authz failures** as security events (user, action, resource) — they're intrusion signals.
- Add timing spans around DB calls in repositories (`drizzle` logger hook or wrapper) → N+1 shows up in
  logs before users feel it (Module 27).

## 29.3 Server timing → traces without a vendor on day one `[SERVER]`

```ts
// lightweight stage timing, surfaced as headers in non-production:
const marks = { mw: 4, route: 11, loaders: 87, render: 23 }  // collected via ctx
// production: export to OTLP (Sentry/Honeycomb/Datadog all accept it)
```

The stages mirror Module 01's lifecycle — when a page is slow, the breakdown tells you *which station* of
the pipeline to fix. Route-level loader timing (per route id) is the highest-value chart in the whole app.

## 29.4 Dashboards worth having (few, decision-grade)

1. p50/p95 SSR time **per route** (regression alarm).
2. Error rate by class (401/403/422/5xx) — 5xx trends page humans; 4xx trends inform product.
3. DB query p95 + slow-query list.
4. Queue depth + job failure rate (Module 26).
5. Webhook signature failures + replay counts.

## 29.5 Environment variables — the complete model

| Category | Prefix/loc | Example | Rules |
|---|---|---|---|
| **Server secrets** | plain env, server process | `DATABASE_URL`, `BETTER_AUTH_SECRET`, `STRIPE_SECRET_KEY` | Never in git; secret store in prod; validated at boot (Module 17.2) |
| **Server config** | plain env | `LOG_LEVEL`, `QUEUE_URL` | Same handling, lower sensitivity |
| **Public client vars** | `VITE_*` | `VITE_APP_PUBLIC_URL` | **Inlined into the client bundle at build time** — treat as public forever |
| **Build-time only** | CI vars | registry tokens | Never referenced by app code |

```ts
// [SERVER] — src/server/env.server.ts (Module 17.2): fail fast, one place reads env
export const env = EnvSchema.parse(process.env)

// [BOTH] — client reads public vars via import.meta.env.VITE_* ONLY
```

### Dangerous mistakes (the hall of shame)

1. `VITE_DATABASE_URL=…` “temporarily” → the connection string ships to every browser.
2. Secrets in a shared module that a component imports → bundler drag into client chunk (import protection
   is your alarm system — Module 01).
3. `.env` committed “just for dev” with staging credentials → rotate immediately, purge history.
4. Logging `process.env` on boot “to debug config” → secrets in log vendor.
5. Runtime vs build-time confusion: setting a `VITE_*` var at *deploy* time does nothing (it was inlined
   at build). Runtime-switchable values must be server-fetched or runtime env on the server.

## 29.6 Anti-patterns

⚠️ **DO NOT** `console.log` objects ad-hoc in production paths — one structured logger, one format.

⚠️ **DO NOT** alert on individual errors; alert on rates and SLO burn.

⚠️ **DO NOT** put secrets in the client “because it's internal anyway” — it's public the moment it ships.

## 29.7 Exercises

- **Beginner:** Ship the observability middleware; verify redaction with a cookie-heavy request.
- **Intermediate:** Per-route loader timing table in dev overlay; find your slowest route and fix it (Module 27).
- **Production:** Sentry integration with `requestId` linking: log line → trace → error, one click.
- **Architecture Challenge:** Design the on-call severity ladder: which of the five dashboards pages a human at 3am, which waits for morning, which never pages?
- **Debug Challenge:** “Production broke because a secret was only set in the build stage.” Using 29.5's table, classify the failure and write the startup check that would have caught it.

🧠 **MENTAL MODEL — Observability:** every request gets an id, every stage gets a timer, every error gets a
class, every secret gets a vault. If you can't answer “which stage, which tenant, which release” in one
query — you're not done.

---

**Next: [Module 30 — Deployment, Runtimes, CI/CD & Production Migrations →](30-deployment.md)**
