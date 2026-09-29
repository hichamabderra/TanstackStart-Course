# Module 27 — Performance Engineering

> The discipline: **MEASURE → IDENTIFY BOTTLENECK → OPTIMIZE → MEASURE AGAIN.** Every technique below is
> tagged with which phase it belongs to. Optimization without a measurement is superstition.

---

## 27.1 The performance budget (define before you tune)

| Metric | Target (Meridian) | Tool |
|---|---|---|
| TTFB (SSR start) | < 300ms | server timing (Module 29) |
| First Contentful Paint | < 1.2s (fast 4G) | Lighthouse / WebPageTest |
| Largest Contentful Paint | < 2.5s | CrUX / RUM |
| Hydration → interactive | < 100ms after FCP | Profiler |
| Client navigation | < 150ms perceived | preloading + cache |
| JS shipped per route | budgeted kB, tracked in CI | bundle analyzer |

## 27.2 The Start performance stack (in order of leverage)

1. **Preloading** (`defaultPreload: 'intent'`, Module 07) — perceived latency → ~0 for warm paths.
2. **Code splitting** (`.lazy.tsx` routes, Module 03) — ship only the current screen's code.
3. **Streaming** (Module 09) — first paint independent of slowest query.
4. **Caching** — router `staleTime` (07) + Query keys (18): don't fetch what you know.
5. **Database** — indexes & query shape (16/27.5): the slowest wins you'll ever ship.
6. **Payload hygiene** — select columns, paginate, compress (27.6).
7. **Virtualization** (21) — only where rendering is the measured cost.

## 27.3 Client-side wins `[CLIENT]`

```bash
npm i -D rollup-plugin-visualizer   # where did the kilobytes go?
```

- **Audit the graph:** visualize the client bundle; hunt chart libs, icon packs, date libs imported whole.
  (Dayjs vs `Intl`; dynamic-import chart libraries inside `.lazy` routes.)
- **Dynamic import heavy widgets:** `const Chart = lazy(() => import('./chart'))` inside Suspense.
- **Memoize rows/cells** in big tables; stable callbacks (`useServerFn` helps).
- **`placeholderData` + skeletons** keep perceived speed high while the next page loads (Module 20).
- **Reduce hydration cost:** fewer serialized bytes in the document (trim loader payloads to what the first
  paint needs; defer the rest via streaming).

## 27.4 Server-side wins `[SERVER]`

- **Parallel loaders / `Promise.all`** — no accidental waterfalls (Module 06).
- **Stream slow widgets** instead of blocking the document (Module 09).
- **Cache headers with intent:** static assets immutable+hashed; SSR HTML `private, no-store` for
  personalized pages (or short `s-maxage` + Vary for public catalog pages — Module 25 warns about tenant
  leaks through shared caches).
- **Server function granularity:** prefer one function returning a screen's batched payload over 6 serial
  round-trips — but don't build god-functions that defeat cache keys (Module 10 architecture exercise).

## 27.5 Database wins (biggest ROI per hour) `[SERVER]`

1. `EXPLAIN ANALYZE` the hot five queries; seek index scans.
2. Composite indexes matching `WHERE orgId = ? ORDER BY x` (Module 16).
3. **Kill N+1:** relations `with:` or `inArray` batching. Detection: log query count per request in dev
   middleware; a list page should issue O(1) queries, not O(rows).
4. Keyset pagination past ~100k rows (Module 20 sidebar).
5. Pool sizing: instances × pool ≤ DB limit; measure wait times.

## 27.6 Network hygiene

- HTTP/2+ everywhere (hosts do this); gzip/brotli on dynamic responses.
- Images: modern formats, `srcset`, lazy-load below the fold; object-storage + CDN (Module 26).
- Avoid chatty RPC: batch reads; keep mutations fine-grained.
- `preload` link hints for critical fonts only if self-hosted (Tailwind default stacks often need none).

## 27.7 A worked optimization loop (the method, applied)

```
SYMPTOM: /dashboard LCP 4.1s in production.
MEASURE: server timing → analytics query 2.6s; client: single 900kB chunk; hydration fine.
IDENTIFY: (1) blocking query, (2) bundle weight.
OPTIMIZE:
  → stream analytics behind Suspense (shell at ~600ms)
  → add (orgId, createdAt) index → query 2.6s → 90ms
  → move chart lib into analytics.lazy.tsx → main chunk 900kB → 380kB
MEASURE AGAIN: LCP 1.7s. Budget met. Ship notes; move on.
```

## 27.8 Anti-patterns

⚠️ **DO NOT** memoize everything “to be fast” — memoization is code with memory cost; measure first.

⚠️ **DO NOT** cache personalized HTML in shared caches without tenant-aware keys/Vary.

⚠️ **DO NOT** optimize the wrong layer (virtualizing a list whose data takes 2s to arrive).

⚠️ **DO NOT** ship a performance fix without the before/after numbers in the PR.

## 27.9 Exercises

- **Beginner:** Run the visualizer on your build; remove or lazy-load your two biggest non-essential dependencies.
- **Intermediate:** Add per-loader server timing; publish a `/dashboard` timing table before/after moving analytics to streaming.
- **Production:** CI performance gate: fail the build if any route chunk exceeds budget; track LCP in staging weekly.
- **Architecture Challenge:** Design the caching strategy for a public catalog page (CDN-friendly) vs the private dashboard (never shared) — document headers + invalidation for each.
- **Debug Challenge:** “App is fast locally, slow in prod.” Enumerate the usual suspects (missing indexes at scale, cold pools, no compression, region latency to DB) and the one command/probe that confirms each.

🧠 **MENTAL MODEL — Performance:** latency hides in *serial waits* and *bytes*. Parallelize, stream, cache,
split, index — in that order of leverage — and always close the loop with a second measurement.

---

**Next: [Module 28 — Testing Strategy: Vitest, RTL, MSW, Playwright →](28-testing.md)**
