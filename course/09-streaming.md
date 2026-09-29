# Module 09 — Streaming SSR with Suspense

> Capstone Stage 14 lives here. Streaming is the difference between “the dashboard appears in 2.5s” and
> “the shell appears in 300ms and each widget lands as it's ready”. Stable, core Start functionality.

---

## 9.1 The problem streaming solves

Classic SSR is all-or-nothing: the server waits for **every** loader before sending **any** HTML. One slow
query (analytics: 1.8s) holds the whole page hostage.

Streaming inverts this: the server sends HTML **as soon as parts are ready**, using React Suspense as the
flush boundary:

```
t=0ms    shell + sidebar + skeletons ──────────────▶ browser paints
t=220ms  summary widget resolves      ─────────────▶ streamed in, Suspense swaps skeleton
t=610ms  recent orders resolve        ─────────────▶ streamed in
t=1850ms analytics resolves           ─────────────▶ streamed in
```

Perceived performance is dominated by *first useful paint*, not total time. Streaming optimizes exactly that.

## 9.2 The deferred-data pattern: loader returns a PROMISE `[SERVER → CLIENT]`

The route loader can return an **un-awaited promise**; the component wraps it in `<Await>` (from
`@tanstack/react-router`), which suspends. During SSR the shell flushes immediately; the widget's HTML
streams when the promise resolves. The same code streams on the server and works on client navigation.

FILE: `src/routes/_authed/dashboard.tsx`

```tsx
import { Await, createFileRoute } from '@tanstack/react-router'
import { getDashboardSummary, getAnalytics, getRecentOrders, getActivity } from '~/server/dashboard.functions'

export const Route = createFileRoute('/_authed/dashboard')({
  loader: () => {
    // Fire ALL queries in parallel; hand the promises to the UI — do NOT await them here
    return {
      summary: getDashboardSummary(),                          // ~200ms
      orders: getRecentOrders(),                               // ~600ms
      activity: getActivity(),                                 // ~700ms
      analytics: getAnalytics({ data: { range: '7d' } }),      // ~1800ms (the slow one)
    }
  },
  component: Dashboard,
})

function Dashboard() {
  const data = Route.useLoaderData()
  return (
    <div className="grid gap-4 md:grid-cols-2">
      {/* Fast: paints almost immediately */}
      <Suspense fallback={<CardSkeleton title="Summary" />}>
        <Await promise={data.summary}>{(s) => <SummaryCard data={s} />}</Await>
      </Suspense>

      {/* Slow: streams in later — user already sees the rest of the page */}
      <Suspense fallback={<CardSkeleton title="Analytics" />}>
        <Await promise={data.analytics}>{(a) => <AnalyticsPanel data={a} />}</Await>
      </Suspense>

      <Suspense fallback={<CardSkeleton title="Recent orders" />}>
        <Await promise={data.orders}>{(o) => <RecentOrders orders={o} />}</Await>
      </Suspense>

      <Suspense fallback={<CardSkeleton title="Activity" />}>
        <Await promise={data.activity}>{(act) => <ActivityFeed items={act} />}</Await>
      </Suspense>
    </div>
  )
}
```

### Why this is the right decomposition

- **Independent widgets = independent Suspense boundaries.** One slow query never blocks siblings.
- The loader still *coordinates* (one place to see what the page needs); rendering decides *patience*.
- Errors inside a deferred promise surface at the `<Await>` boundary → pair each with an `ErrorBoundary`
  (Module 24) so analytics dying doesn't kill the dashboard.

## 9.3 Streaming SERVER FUNCTIONS — streams over RPC `[SERVER → CLIENT]`

Server functions may return a `Response` — including one backed by a `ReadableStream` — letting the server
push bytes progressively (live logs, token streams, CSV exports):

FILE: `src/server/exports.functions.ts` `[SERVER]`

```ts
import { createServerFn } from '@tanstack/react-start'

export const streamOrdersCsv = createServerFn({ method: 'GET' }).handler(async () => {
  const stream = new ReadableStream({
    async start(controller) {
      const encoder = new TextEncoder()
      controller.enqueue(encoder.encode('id,total,status\n'))
      for await (const order of iterateOrders()) {   // async generator → repository
        controller.enqueue(encoder.encode(`${order.id},${order.total},${order.status}\n`))
      }
      controller.close()
    },
  })
  return new Response(stream, {
    headers: {
      'content-type': 'text/csv; charset=utf-8',
      'content-disposition': 'attachment; filename="orders.csv"',
    },
  })
})
```

```tsx
// [CLIENT] — consume it: trigger via a link/navigation, or fetch the fn URL
// For UI streaming, read progressively:
const res = await streamOrdersCsv()           // Response rides back over RPC
const reader = res.body!.getReader()
```

Also useful: returning `Response` from a server function lets you set custom headers (downloads, caching
hints) or short-circuit with redirects.

## 9.4 Streaming + auth + cache: the composition rules

1. **Don't defer what the shell needs.** Layout chrome, session-dependent header data → await in the loader.
2. **Per-user data streams fine** — it's per-request SSR, not a shared HTTP cache. (Careful: adding shared
   `Cache-Control` headers to personalized HTML leaks data between users — Module 25.)
3. **Suspense fallbacks must render fast and server-safe** — skeletons only, no client-only APIs.
4. With Query (Module 18), streamed boundaries can alternatively be `useSuspenseQuery` inside client
   components — choose per widget; don't mix two ownership models for one widget.

## 9.5 Case study: Meridian dashboard request lifecycle

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant S as Start (SSR)
  participant SF as Server functions
  participant DB as Postgres
  B->>S: GET /dashboard
  S->>S: _authed.beforeLoad → session ok (Module 14)
  S->>SF: loader fires 4 fns in parallel
  SF->>DB: 4 queries (200ms / 600ms / 700ms / 1800ms)
  S-->>B: shell + 4 skeletons (flush #1, ~250ms)
  DB-->>SF: summary ready → S-->>B: flush #2 (summary HTML)
  DB-->>SF: orders+activity → S-->>B: flush #3
  DB-->>SF: analytics → S-->>B: flush #4
  Note over B: progressive paint; hydration per flush
```

## 9.6 Anti-patterns

⚠️ **DO NOT** await the slow promise in the loader “just to be safe” — you've rebuilt blocking SSR.

⚠️ **DO NOT** wrap the whole page in one Suspense boundary — granularity is the feature.

⚠️ **DO NOT** stream personalized pages through shared CDN caches without vary/bypass rules.

⚠️ **DO NOT** use streaming as a substitute for fixing a 4s query (Module 27: indexes first, streaming second).

> **Verification note:** I could not fetch a standalone “streaming” guide page in the current docs on
> 2026-09-28 (docs reorg); the deferred/`<Await>` pattern above matches the stable Router deferred-data API
> and Start's streaming SSR overview. Re-check the Start docs “Streaming” section when you build this.

## 9.7 Exercises

- **Beginner:** Split your dashboard into 2 widgets with fake delays (500ms/2s). Observe flush order with “Disable cache” + slow network throttling.
- **Intermediate:** Add an `ErrorBoundary` per widget; make the analytics function randomly throw; verify siblings survive.
- **Production:** Stream the CSV export for 100k rows; measure memory on the server (it must stay flat — that's the point of `ReadableStream`).
- **Architecture Challenge:** Which dashboard widgets deserve streaming vs awaiting vs client-side Query? Build the decision table (first-paint importance × latency × personalization).
- **Debug Challenge:** Skeletons flash even though data resolves in 40ms. What router/loader setting or Suspense placement causes visible flicker, and how do you fix it? (Hint: Module 03's `defaultPendingMs`/`defaultPendingMinMs` apply to route-level pending; Suspense has no such grace period — consider whether tiny promises should just be awaited.)

🧠 **MENTAL MODEL — Streaming:** SSR is a *progressive download*, not a single response. Suspense boundaries
are loading docks: whatever's ready ships; the truck doesn't wait for the slowest pallet.

---

**Next: [Module 10 — Server Functions In Depth →](10-server-functions.md)**
