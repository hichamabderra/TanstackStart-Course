# Module 01 — The TanStack Start Mental Model & Execution Model

> If you internalize only two modules, make it this one and Module 07. Everything else in the course is an
> elaboration of the pictures drawn here.

---

## 1.1 The equation

```
    TANSTACK ROUTER                (type-safe routing, loaders, search schemas, cache)
  + SERVER RUNTIME                 (an actual server that runs your code per request)
  + SERVER FUNCTIONS               (type-safe RPC from anywhere in the app to the server)
  + SERVER ROUTES                  (plain HTTP endpoints for the outside world)
  + SSR + STREAMING                (render React on the server, stream it progressively)
  + VITE / RSBUILD BUILD SYSTEM    (client bundle + server bundle + dev server)
  ─────────────────────────────
  = TANSTACK START
```

Read it in this order: **Start is router-first.** The routing model — the route tree, the loaders, the search
schemas, the typed `Link` — is the *application contract*. The server runtime exists to **serve** that model:
it renders routes, executes their loaders, and hosts the functions they call. This inverts the mental model of
frameworks where "the backend" is the center and the UI is bolted on, and it's different from SPAs where the
server is just a JSON vending machine.

> **WHY TANSTACK?** Because the router is type-safe end-to-end, *every* layer that hangs off it inherits the
> types: URL → params → search → loader data → server function inputs/outputs → UI. One contract, zero drift.

## 1.2 The request lifecycle — stage by stage

This is the pipeline for **every** page request. Memorize it.

```
URL
 ↓
1. Route Matching
 ↓
2. beforeLoad / middleware        (auth checks, route context, redirects)
 ↓
3. Loader                         (data coordination for this route)
 ↓
4. Server / Client execution      (loaders may call server functions over RPC)
 ↓
5. Component                      (React renders with loader data)
 ↓
6. SSR / Streaming                (HTML streamed to the browser)
 ↓
7. Hydration                      (React attaches to the streamed HTML)
 ↓
8. Client Navigation              (subsequent navigations skip full-document SSR)
```

### Stage 1 — Route Matching `[SERVER + CLIENT]`

The URL is matched against the **route tree** (generated from your `src/routes/` files). Matching produces the
chain of route objects from root to leaf: e.g. `/products/42?sort=price` → `__root` → `products` →
`products/$productId`. Both the initial server request and later client navigations run matching — the route
tree is *shared code*.

### Stage 2 — `beforeLoad` / middleware `[SERVER] (on first request) / [BOTH] (client nav)`

`beforeLoad` runs **before** the loader. This is where you make routing *decisions*: am I authenticated?
Should I redirect? What context does this branch of the tree need? On the server it runs during SSR; on client
navigations it runs in the browser. **Middleware** (Module 13) is the server-side counterpart that wraps *HTTP*
requests: logging, CSRF, session resolution — before any route code executes.

> ⚠️ **DO NOT** treat `beforeLoad` as a security boundary. It is route **UX**. The server must re-authorize
> every data access (Modules 14/15).

### Stage 3 — Loader `[SERVER on first request → CLIENT on client nav]`

The route loader *coordinates the data this route's UI needs*. Loaders of matched routes run **in parallel**.
A loader can: run a query directly (same process, server context), or call a **server function** — which works
identically whether the loader executes on the server (direct call) or in the browser (RPC).

### Stage 4 — Server / Client execution `[SERVER]`

This is where server functions and server routes execute: database access, secret reads, hashing, third-party
APIs. The output is **serialized** across the boundary — Start type-checks serializability by default.

### Stage 5 — Component `[SERVER then CLIENT]`

React renders route components with loader data. On first load this happens on the server (producing HTML);
after hydration, re-renders happen in the browser.

### Stage 6 — SSR / Streaming `[SERVER → CLIENT]`

The HTML document is rendered and **streamed**: Suspense boundaries let slow data arrive later while the fast
shell is already painted (Module 09).

### Stage 7 — Hydration `[CLIENT]`

The browser receives HTML immediately, then JS arrives, React walks the existing DOM and attaches event
handlers. Dehydrated state (loader data, and Query cache if you use it) is shipped with the document so the
client **doesn't refetch** what the server already fetched.

### Stage 8 — Client Navigation `[CLIENT]`

After hydration, `<Link>` clicks do **not** request a new HTML document. The router matches client-side,
runs `beforeLoad` + loaders in the browser; loaders call server functions via HTTP; React re-renders the diff.
This is the SPA phase of the lifecycle. First request = SSR; everything after = SPA. **Same routes, same
loaders, same components** — that's the point.

## 1.3 SSR vs server-only execution — NOT the same thing

This distinction is a filter question in senior interviews, and it trips up everyone coming from SPAs.

| Concept | Definition | Example in Start |
|---|---|---|
| **Server-side rendering (SSR)** | React components are *rendered to HTML on the server* — so the user sees content fast and crawlers get markup. Code here must be UI-safe: no secrets, no writes, ideally read-only data. | Route `component`, root layout, loader data flowing into HTML |
| **Server-only execution** | Code that *only ever runs in the server process* and whose source may never ship to the browser. It may or may not produce HTML. | Server functions, server route handlers, `*.server.ts` modules, password hashing, DB client |

```
            ┌────────────────────────── SERVER PROCESS ──────────────────────────┐
            │                                                                    │
  request ─▶│  middleware → router → loaders → React render → HTML stream        │  ← SSR
            │                                                                    │
            │  server functions: hash password, query DB, read secrets           │  ← server-only
            │  server routes:    /api/webhooks/stripe, /rss.xml                  │    (no HTML involved)
            └────────────────────────────────────────────────────────────────────┘
```

A component can be SSR'd and still be "client-owned" after hydration; a server function is never rendered at
all. **SSR is about HTML. Server-only execution is about trust.** Keep the two axes separate in your head and
the whole framework becomes predictable.

## 1.4 The execution map — where everything runs

Every example in this course carries one of these labels:

- `[SERVER]` — only ever in the server process
- `[CLIENT]` — only ever in the browser
- `[SERVER → CLIENT]` — computed on server, serialized to the client
- `[BOTH]` — shared code that runs in both environments (must be environment-safe)

| Building block | Runs | Notes |
|---|---|---|
| Route matching & route tree | `[BOTH]` | Shared; types shared too |
| `beforeLoad` | `[BOTH]` (server during SSR, browser on client nav) | Never a security boundary |
| Route `loader` | `[BOTH]` | Server during SSR; browser during client nav (then usually calls server fns) |
| Server Function handler | `[SERVER]` | Invoked from client or server; RPC-stubbed in client bundle |
| Server Route handler | `[SERVER]` | Plain HTTP; externally callable |
| Middleware `.server()` | `[SERVER]` | Wraps SSR, server fns, server routes |
| Function-middleware `.client()` | `[CLIENT]` | Wraps the RPC call on the client side |
| Route component | `[SERVER → CLIENT]` | Rendered server-first, then hydrated |
| Suspense fallback | `[BOTH]` | Painted during streaming AND client nav |
| Cookies util (`getCookie`/`setCookie`) | `[SERVER]` (request path) | Client-side cookie reads are a different tool |
| `createIsomorphicFn()` | `[BOTH]` — branch per env | Tree-shaken per bundle |
| `createServerOnlyFn()` | `[SERVER]` | Throws if somehow called in client |
| Browser APIs (`localStorage`, `matchMedia`) | `[CLIENT]` | Guard with effects / client-only code |
| Zod schemas, shared types | `[BOTH]` | The whole point: one source of truth |
| Drizzle `db` client | `[SERVER]` | Must live behind `*.server.ts` / import protection |

### Environment functions (verified current API)

```ts
// [BOTH] — src/lib/env.ts
import { createIsomorphicFn, createServerOnlyFn, createClientOnlyFn } from '@tanstack/react-start'

// Adapts per environment; the other branch is tree-shaken out of each bundle
export const getUserId = createIsomorphicFn()
  .server(() => readSessionUserIdFromRequest())   // [SERVER]
  .client(() => undefined)                        // [CLIENT]

// Server-only: calling it in the browser throws at runtime, and its
// module graph is removed from the client bundle at build time
export const hashPassword = createServerOnlyFn(async (pw: string) => {
  return await crypto.subtle.digest(/* ... */)
})
```

And **import protection**: files matching `**/*.server.*` cannot be imported from client code — the build
fails with a trace telling you to wrap the logic in a server function instead. You can also mark any module
server-only explicitly:

```ts
// [SERVER] — src/lib/secrets.ts
import '@tanstack/react-start/server-only'
export const STRIPE_SECRET = process.env.STRIPE_SECRET_KEY
```

## 1.5 The two lives of one codebase

```
BUILD TIME
  vite build
    ├─ client bundle  → browser   (components, router, RPC stubs, no server code)
    └─ server bundle  → runtime   (SSR, server fns, server routes, db, secrets)

RUN TIME
  First request:   browser ←── HTML stream ──── server renders router+loaders
  Client nav:      browser ──RPC──▶ server fns ──▶ db        (SPA mode)
  External call:   3rd party ──HTTP──▶ server routes         (API mode)
```

🧠 **MENTAL MODEL — "One contract, two bundles, three entry paths."**
The route tree is the contract. Vite splits it into a client bundle and a server bundle. At runtime there are
three paths into the server: (1) SSR of a route, (2) RPC of a server function, (3) HTTP of a server route.
Every design question in this course reduces to: *which path does this feature travel, and where does it run?*

## 1.6 Where this differs from what you know

| If you know… | The adjustment |
|---|---|
| **Next.js App Router** | Start has **no** `'use client'`/`'use server'` string pragmas; the server boundary is an explicit API (`createServerFn`, server routes, `.server.ts` files). Data loading is *route-level and typed*, not scattered through async components (RSC aside, and that's experimental here). |
| **Remix / React Router 7** | Closest cousin: loaders + actions. Start adds type-safe search schemas, the router cache, typed RPC server functions, and deployment flexibility beyond the adapter model. |
| **SPA + REST API** | No separate backend repo to keep in sync: server functions ARE the backend boundary, and they're *typed across the wire*. |
| **tRPC** | Server functions are similar in spirit (typed RPC) but live inside the framework: they compose with middleware, cache invalidation, and the request lifecycle. |

## 1.7 Exercises

- **Beginner:** Draw (on paper!) the 8-stage lifecycle for `GET /products/42?sort=price`. Mark each stage `[SERVER]` or `[CLIENT]` for (a) the first request and (b) a subsequent client navigation.
- **Intermediate:** In one paragraph each, explain to a teammate: “What's the difference between SSR and a server function?” Use no jargon. If you can't, re-read 1.3.
- **Architecture Challenge:** Your app needs: (a) a PDF invoice download, (b) a “new order” email, (c) the order total shown in the page. Assign each to SSR / server function / server route and justify.
- **Debug mindset:** A teammate says “my `console.log` in the loader prints twice.” Using the lifecycle, hypothesize where each print ran.

---

**Next: [Module 02 — Project Setup: CLI, Vite, Tailwind v4, Anatomy →](02-project-setup.md)**
