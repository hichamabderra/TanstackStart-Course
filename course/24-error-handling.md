# Module 24 — Error Handling: Every Layer, 401 vs 403, Four States

> Production apps don't avoid errors; they *classify* them. This module gives Meridian one error vocabulary
> from PostgreSQL to the pixel layer.

---

## 24.1 The taxonomy (learn it once, use it everywhere)

| Class | Examples | Status | Client treatment |
|---|---|---|---|
| **Validation** | bad input, schema failure | 422 | field-level errors (Module 19) |
| **Authentication** | no/expired session | **401** | redirect to login with `?redirect=` |
| **Authorization** | authenticated but not allowed | **403** | “not allowed” state (stay put) |
| **Not found** | missing/hidden resource | 404 | route `notFoundComponent` |
| **Conflict** | duplicate slug, version clash, insufficient stock | 409 | actionable message |
| **Rate limited** | too many attempts | 429 | retry-after guidance |
| **Unexpected** | bug, dependency down | 500 | generic message + retry; full details server-side only |

### 401 vs 403 — the interview question, operationalized

```
401 → “I don't know who you are.”      → log in (identity missing/expired)
403 → “I know you — you can't.”       → do not send to login; show forbidden
```

Consequences: a **401 mid-session** means session expiry — recover by routing to login and coming back.
A **403** means policy worked — logging out won't help; fix the permission or move on. Conflating them
causes login loops and confused users.

## 24.2 Server side: one error vocabulary `[SERVER]`

FILE: `src/server/errors.ts`

```ts
export class AppError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string,
    public details?: unknown,
  ) { super(message) }
}
export const Unauthorized  = (msg = 'Authentication required') => new AppError(401, 'UNAUTHORIZED', msg)
export const Forbidden     = (msg = 'Not allowed')             => new AppError(403, 'FORBIDDEN', msg)
export const NotFound      = (msg = 'Not found')               => new AppError(404, 'NOT_FOUND', msg)
export const Conflict      = (msg: string)                     => new AppError(409, 'CONFLICT', msg)
export const RateLimited   = (retryAfter: number)              => new AppError(429, 'RATE_LIMITED', 'Slow down', { retryAfter })
```

- **Server functions:** throw these from handlers. Start rejects the caller's promise; check your Start
  version's error guide for the exact wire representation (`HttpError`/status passthrough) and map
  consistently in one client-side helper (24.4).
- **Server routes:** catch into the standard envelope (Module 12):

```ts
try { return await handle(request) }
catch (err) {
  if (err instanceof AppError) return jsonError(err.status, err)   // expected → shaped
  logger.error({ err, requestId })                                  // unexpected → logged, generic 500 out
  return jsonError(500)
}
```

- **Database layer:** translate constraint errors at the repository edge:

```ts
catch (e) {
  if (isUniqueViolation(e, 'products_org_slug_uq')) throw Conflict('Slug already in use')
  if (isFkViolation(e)) throw Conflict('Referenced record no longer exists')
  throw e
}
```

**Rule:** expected errors carry *user-safe* messages; unexpected errors keep details **server-side only**.

## 24.3 Route level: boundaries where they belong `[SERVER → CLIENT]`

```tsx
// __root.tsx — LAST-RESORT boundary
export const Route = createRootRoute({
  errorComponent: ({ error }) => {
    // Never render error.message raw for 5xx (may contain internals)
    return <FatalErrorScreen retryable={false} />
  },
  notFoundComponent: () => <GlobalNotFound />,
})

// Feature route — graceful degradation
export const Route = createFileRoute('/_authed/orders/$orderId')({
  errorComponent: ({ error, reset }) => {
    const cls = classify(error)                 // 24.4
    if (cls.status === 401) return null         // global handler redirected already
    if (cls.status === 404) return <OrderNotFound />
    return <InlineError onRetry={reset} message={cls.userMessage} />
  },
})
```

Components with their own async life (Query widgets) also get **React Error Boundaries** so one dead widget
doesn't sink the page (Module 09's dashboard).

## 24.4 Client side: one classifier, one recovery policy `[CLIENT]`

FILE: `src/lib/api-errors.ts`

```ts
export type ApiErrorInfo = { status: number; code: string; userMessage: string; retryable: boolean }

export function classify(err: unknown): ApiErrorInfo { /* parse fn/route errors → info */ }

// Global policy, applied once (Query defaults + call sites):
export function onApiError(info: ApiErrorInfo) {
  switch (info.status) {
    case 401:
      queryClient.clear(); router.invalidate()
      throw redirect({ to: '/login', search: { redirect: location.href } })
    case 403: toast.error("You don't have access to that."); break
    case 429: toast.warning(`Too many attempts — retry in ${info.details.retryAfter}s`); break
    case 500: case 503: toast.error('Something went wrong on our side.'); break  // retryable
  }
}
```

Query integration:

```ts
new QueryClient({
  defaultOptions: {
    queries: { retry: (count, err) => classify(err).retryable && count < 2 }, // don't retry 4xx
  },
})
```

## 24.5 Network layer `[CLIENT]`

- **Retry policy:** retry idempotent reads with backoff (Query default with our classifier); never blind-retry
  POSTs (idempotency keys — Module 12 — make it safe when needed).
- **Offline/abort:** navigations cancel in-flight queries automatically; surface “reconnected” states for
  polled widgets.
- **Timeouts:** long server functions (imports, reports) should be jobs (Module 26), not 30s fetches.

## 24.6 The four-state contract (recap from Module 06/23)

Every data surface: `loading` skeleton → `empty` CTA → `error` (classified message + retry when retryable)
→ `success`. SSR renders whichever state is known at request time; client transitions between them.

## 24.7 Logging & reporting `[SERVER]`

- Expected errors (4xx): **info-level** log (rate-limit hits, bad input) — not pager material.
- Unexpected (5xx): **error-level** with `requestId`, route/fn name, sanitized context (Module 29); ship to
  error tracking (Sentry et al.) with release + user-tenant context.
- Include `requestId` in every response (`x-request-id`) so users can hand it to support.

## 24.8 Anti-patterns

⚠️ **DO NOT** `catch {}` and render nothing — silent failure is the worst UX *and* the hardest bug.

⚠️ **DO NOT** send `error.stack` or DB messages to the browser. Ever.

⚠️ **DO NOT** retry mutations by default; retry reads selectively.

⚠️ **DO NOT** turn every expected 4xx into a toast storm — map once, globally (24.4).

## 24.9 Exercises

- **Beginner:** Implement `errors.ts` + classifier + toasts; force a 422, 401, 403, 404, 500 with test endpoints and verify each treatment.
- **Intermediate:** Session-expiry flow: 401 during a mutation → login redirect → return → form values preserved (Module 19's “never clear work”).
- **Production:** Wire Sentry with `requestId` correlation + PII scrubbing; alert on 5xx rate, not individual errors.
- **Architecture Challenge:** Design error behavior for streaming widgets (Module 09): which failures degrade one widget vs the whole page vs redirect?
- **Debug Challenge:** Users see “Unauthorized” toasts randomly while typing in a search box. Trace: keystroke → URL → Query refetch → stale session → 401 → global handler. What's the humane recovery?

🧠 **MENTAL MODEL — Errors:** every error has a *class*, and every class has exactly one *policy*. Classify
once at the boundary; render, retry, redirect, or alert by class — never improvising per screen.

---

**Next: [Module 25 — Security: The Full Production Threat Model →](25-security.md)**
