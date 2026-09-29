# Module 28 — Testing Strategy: Vitest, RTL, MSW, Playwright

> Capstone Stage 16. A full-stack framework needs a full-stack test plan. Verified current majors:
> **Vitest 5**, **Playwright 1.63**, **MSW 3** (all newer majors than most tutorials — APIs below match
> the current docs; re-check flags when you install).

---

## 28.1 The pyramid for a Start app

```
            E2E (Playwright)          few, critical journeys: signup→login→order→admin action
        Integration (Vitest + real DB) server flows: service→repo→Postgres; routes via fetch()
      Component (Vitest + RTL)         forms, tables, state switches — behavior, not pixels
    Unit (Vitest)                      policies, schemas, mappers, price math — the pure core
```

Ratio intuition for Meridian: **~70% unit, ~20% integration, ~10% E2E** — but weight by risk: authz and
money code skew integration-heavy.

## 28.2 Unit — the pure core `[NODE]`

FILE: `src/server/authz/policies.test.ts`

```ts
import { describe, expect, it } from 'vitest'
import { canViewOrder, hasPermission } from './policies'

describe('canViewOrder', () => {
  const alice = { id: 'u1', role: 'USER', orgId: 'orgA' } as SessionUser
  it('owner can view own order', () => {
    expect(canViewOrder(alice, order({ orgId: 'orgA', userId: 'u1' }))).toBe(true)
  })
  it('cannot view another tenant order', () => {
    expect(canViewOrder(alice, order({ orgId: 'orgB', userId: 'u2' }))).toBe(false)
  })
  it('admin with orders:manage can view any order', () => {
    expect(canViewOrder(admin, order({ orgId: 'orgB' }))).toBe(true)
  })
})
```

Also unit-test: Zod schemas (accept valid, reject + classify invalid), pricing math, id mapping. These tests
are seconds-fast and catch regressions at commit time.

## 28.3 Integration — real database, real flows `[NODE]`

```ts
// vitest.config.ts — integration project points at a disposable test database
// (docker compose postgres; per-file schema, or transaction rollback per test)

import { db } from '~/server/db/index.server'

it('listProducts scopes by org and respects search', async () => {
  await seed({ orgA: [product({ name: 'Laptop Pro' })], orgB: [product({ name: 'Laptop Air' })] })
  const { rows } = await listProductsForOrg('orgA', { q: 'laptop', page: 1, pageSize: 20 })
  expect(rows.map(p => p.name)).toEqual(['Laptop Pro'])   // orgB invisible — tenant test!
})

it('placeOrder decrements stock atomically', async () => { /* Module 16.5 concurrency test */ })
```

**Server routes without a browser** — call the fetch handler directly:

```ts
import serverEntry from '../src/server'   // createServerEntry default export

it('webhook rejects bad signature', async () => {
  const res = await serverEntry.fetch(new Request('http://test/api/webhooks/stripe', {
    method: 'POST', body: '{}', headers: { 'stripe-signature': 'bogus' },
  }))
  expect(res.status).toBe(400)
})
```

**Cookie flows:** build requests with `headers: { cookie: 'session=…' }` from a seeded session — login,
then call protected routes; assert 401/403 paths (Module 24's taxonomy, tested).

## 28.4 Component — RTL on the interactive surface `[JSdom]`

```tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

it('login form maps server 401 to a global error and keeps values', async () => {
  mockLoginFn.mockRejectedValueOnce(ApiError401)
  render(<LoginPage />)
  await userEvent.type(screen.getByLabelText(/email/i), 'sam@example.com')
  await userEvent.type(screen.getByLabelText(/password/i), 'hunter22')
  await userEvent.click(screen.getByRole('button', { name: /sign in/i }))
  expect(await screen.findByRole('alert')).toHaveTextContent(/check your credentials/i)
  expect(screen.getByLabelText(/email/i)).toHaveValue('sam@example.com') // work preserved (Module 19)
})
```

Test **behavior through roles** (`getByRole`), not class names. Cover the four states: loading skeleton →
empty → error → success (Module 23).

## 28.5 Mocking with MSW 3 — where mocks belong `[NODE + BROWSER]`

```ts
// test/mocks/handlers.ts — external services ONLY
import { http, HttpResponse } from 'msw'
export const handlers = [
  http.post('https://api.stripe.com/v1/payment_intents', () =>
    HttpResponse.json({ id: 'pi_test', status: 'succeeded' })),
  http.post('https://email.example.com/send', () => HttpResponse.json({ ok: true })),
]
```

Decision rule:

| Boundary | Mock or real? |
|---|---|
| External SaaS (payments, email, maps) | **Mock (MSW)** — unstable/expensive/third-party |
| Your Postgres | **Real** (integration DB) — SQL is the thing under test |
| Your server functions from components | **Mock** in component tests; **real** in E2E |
| Time/random/ids | Fake (`vi.useFakeTimers`, injected ids) — determinism |

Mocking *your own* database in integration tests defeats the purpose — the query IS the feature.

## 28.6 E2E — Playwright on the real stack `[BROWSER]`

```ts
// e2e/orders.spec.ts
test('user can place an order and sees it in history', async ({ page }) => {
  await registerAndLogin(page, 'user@orgA.test')
  await page.goto('/products')
  await page.getByRole('link', { name: 'Laptop Pro' }).click()
  await page.getByRole('button', { name: 'Add to cart' }).click()
  await page.goto('/checkout')
  await page.getByRole('button', { name: 'Place order' }).click()
  await expect(page.getByText('Order confirmed')).toBeVisible()
  await page.goto('/orders')
  await expect(page.getByRole('row', { name: /Laptop Pro/ })).toBeVisible()
})

test('cross-tenant access is refused', async ({ page }) => {
  await loginAsOrgAUser(page)
  const res = await page.request.get(`/api/v1/orders/${orgBOrderId}`)
  expect([403, 404]).toContain(res.status())
})
```

E2E list for Meridian: signup→verify→login; place order; admin edits product; cross-tenant 403 suite;
optimistic favorite + forced-error rollback; 401 mid-session recovery.

Also valuable: **axe-core in Playwright** (`@axe-core/playwright`) on every screen — a11y regressions fail CI.

## 28.7 Test-data & environments

- `docker compose up db-test` (Postgres 16); Vitest integration project runs migrations on setup.
- Seed builders (`seedUser({ orgRole })`) over raw fixtures — keep schema changes cheap.
- Never let integration tests touch dev/staging DBs; one env var decides (`DATABASE_URL_TEST`).

## 28.8 Anti-patterns

⚠️ **DO NOT** E2E-test validation messages the unit suite already owns — slow and brittle.

⚠️ **DO NOT** mock the DB under “integration” tests — you're testing a ghost.

⚠️ **DO NOT** write snapshot tests for HTML you'll change weekly — assert semantics instead.

⚠️ **DO NOT** let E2E depend on wall-clock time or shared mutable data — seed per test, fake the clock.

## 28.9 Exercises

- **Beginner:** Unit-test your policy module + two Zod schemas to 100% branch coverage.
- **Intermediate:** Integration-test `placeOrder` incl. the stock-conflict case against docker Postgres.
- **Production:** Full Playwright suite for the five journeys in 28.6 + axe scans; wire into CI (Module 30).
- **Architecture Challenge:** Design test isolation for parallel Vitest workers sharing one Postgres (schemas per worker vs DB per worker) — pick one, justify by CI time.
- **Debug Challenge:** An E2E test passes locally, fails in CI 10% of the time. List the classic causes (seed race, animation timing, port reuse, timezones) and a fix protocol.

🧠 **MENTAL MODEL — Testing:** unit tests own the *rules*, integration tests own the *flows*, E2E owns the
*journeys*, mocks stand in for *other people's systems only* — and the database is always real when SQL is
the point.

---

**Next: [Module 29 — Observability & Environment Variables →](29-observability-env.md)**
