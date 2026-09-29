# Module 31 — Start vs Next.js vs React Router · Experimental Features · Official Docs Map

> No marketing allowed in this module. Every claim is a tradeoff you can act on.

---

## 31.1 TANSTACK START vs NEXT.JS (2026, honestly)

| Dimension | TanStack Start (RC, 1.168.x) | Next.js (mature) |
|---|---|---|
| **Routing** | Route tree from files + **code-based equally first-class**; fully typed params/search/loaders | File-system conventions; weaker static typing of route ids/params |
| **Type safety** | End-to-end: route ids, params, search schemas, loader data, RPC inputs/outputs | Improving (typedRoutes etc.) but not the core design principle |
| **Data loading** | Route loaders + router cache (+ optional Query); explicit | Async Server Components / fetch-cache; historically shifting semantics |
| **SSR/streaming** | Full-document SSR + Suspense streaming (stable) | Equivalent capability, more mature in the wild |
| **Server mutations** | `createServerFn`: typed RPC, middleware-composable | Server Actions: similar ergonomics, different boundary model |
| **Server routes** | `server.handlers` in the same route tree | Route Handlers (stable, long-lived) |
| **Middleware** | `createMiddleware` (request + function), runs with your server runtime | Edge middleware with its own runtime constraints |
| **RSC** | **Experimental** (opt-in) | **Core, production-proven** — Next's center of gravity |
| **Deployment** | Bring-your-own: Workers, Netlify, Railway, Nitro targets, Node, Bun | First-class Vercel; other hosts via adapters (workable, more friction) |
| **Ecosystem** | Younger, growing fast; official partner integrations | Massive (auth, CMS, UI kits all Next-first) |
| **Lock-in** | Low: standard Vite build, fetch-handler server, portable | Higher: framework-specific build/runtime features |
| **Maturity** | RC line, moving fast, docs improving | Years of production hardening |

### Practical verdict

- **Choose Start** for data-heavy, type-safety-critical apps where you want explicit control over caching,
  deployment, and the server boundary — and you accept RC momentum.
- **Choose Next.js** when you need mature RSC today, the largest ecosystem, or your team/hosting is already
  Vercel-centered.
- **Migration note:** Start ships an official *Migrate from Next.js* guide and a comparison page — read both
  before quoting timelines to anyone.

## 31.2 React Router vs TanStack Router vs TanStack Start

| | React Router | TanStack Router | TanStack Start |
|---|---|---|---|
| Category | Router (+ “framework mode”) | Router (SPA-first, type-safe) | **Full-stack framework** |
| Data loading | loaders/actions in framework mode | loaders + built-in cache | + SSR/streaming/server runtime |
| Type safety | partial | **deep by design** | inherits Router |
| Server story | bring your own / framework mode | none | server functions, server routes, middleware |
| When | existing RR codebases, Remix heritage | great SPA needing best-in-class routing | the full platform this course builds on |

The distinction that clarifies every debate: **a router navigates between screens; a framework owns the
request.** Start is Router + the server it deserves.

## 31.3 CURRENTLY EXPERIMENTAL — the labeled list

### React Server Components in Start ⚠️

Verified facts (2026-09-28): docs banner **“Server Components are experimental! The API may see
refinements.”** Opt-in: `@vitejs/plugin-rsc` + `tanstackStart({ rsc: { enabled: true } })`; requires
React 19+, Vite 7+ (or Rsbuild 2+). Backed by the separate `@tanstack/react-start-rsc` package at `0.1.x`.

The current shape: server-rendered UI returned through server functions and surfaced via
`renderServerComponent(...)` (renderable values) and `createCompositeComponent(...)` +
`<CompositeComponent src={...} />` (slots: children / render props / component props).

**Why it exists:** smaller bundles (heavy libs stay server-side), colocated data fetching, secure-by-default
server logic, progressive streaming.

**How it differs from standard SSR:** SSR renders your *client* component tree on the server and then
hydrates that same code in the browser. RSC renders a *server* component tree whose output (serialized UI)
streams to the client — the server components' JS never ships.

**Risks:** API churn (0.x), serialization pitfalls at slots, tooling/eco immaturity, debugging complexity.

**When to experiment:** content-heavy pages with heavy rendering deps; isolated widgets behind a boundary.
**When NOT to use in production:** as your foundation, in regulated change-control environments, or when you
can't absorb breaking churn. The course capstone deliberately does not depend on RSC.

### Other watch-items

- **Rsbuild support** — officially supported build path (newer than Vite); expect rough edges at the
  boundaries (verify before betting a release on it).
- **Evolving deployment integrations** — partner plugins move fast; re-read the hosting guide each upgrade.
- Anything exported with `unstable_`/`experimental` prefixes — treat as exactly what the name says.

## 31.4 FINAL DOCUMENTATION MAP (official links only)

**TanStack Start** — [Overview](https://tanstack.com/start/latest/docs/framework/react/overview) ·
[Getting Started](https://tanstack.com/start/latest/docs/framework/react/getting-started) ·
[Build from scratch](https://tanstack.com/start/latest/docs/framework/react/build-from-scratch) ·
[Server Functions](https://tanstack.com/start/latest/docs/framework/react/guide/server-functions) ·
[Server Routes](https://tanstack.com/start/latest/docs/framework/react/guide/server-routes) ·
[Middleware](https://tanstack.com/start/latest/docs/framework/react/guide/middleware) ·
[Server Entry Point](https://tanstack.com/start/latest/docs/framework/react/guide/server-entry-point) ·
[Server Components (experimental)](https://tanstack.com/start/latest/docs/framework/react/guide/server-components) ·
[Hosting](https://tanstack.com/start/latest/docs/framework/react/guide/hosting) ·
[Environment Functions](https://tanstack.com/start/latest/docs/framework/react/guide/environment-functions) ·
[Import Protection](https://tanstack.com/start/latest/docs/framework/react/guide/import-protection) ·
[Start vs Next.js](https://tanstack.com/start/latest/docs/framework/react/start-vs-nextjs) ·
[v1 RC announcement](https://tanstack.com/blog/announcing-tanstack-start-v1)

**TanStack Router** — [Overview](https://tanstack.com/router/latest/docs/framework/react/overview) ·
[File-based Routing](https://tanstack.com/router/latest/docs/framework/react/routing/file-based-routing) ·
[Data Loading](https://tanstack.com/router/latest/docs/framework/react/guide/data-loading) ·
[Preloading](https://tanstack.com/router/latest/docs/framework/react/guide/preloading) ·
[Search Params](https://tanstack.com/router/latest/docs/framework/react/guide/search-params) ·
[Deferred Data](https://tanstack.com/router/latest/docs/framework/react/guide/deferred-data-loading)

**TanStack Query** — [Overview](https://tanstack.com/query/latest/docs/framework/react/overview) ·
[SSR & Hydration](https://tanstack.com/query/latest/docs/framework/react/guide/ssr) ·
[Optimistic Updates](https://tanstack.com/query/latest/docs/framework/react/guide/optimistic-updates) ·
[Infinite Queries](https://tanstack.com/query/latest/docs/framework/react/guide/infinite-queries)

**TanStack Form** — [Overview](https://tanstack.com/form/latest/docs/framework/react/overview) ·
[Validation](https://tanstack.com/form/latest/docs/framework/react/guide/validation)
**Table** — [Docs](https://tanstack.com/table/latest/docs/introduction) ·
**Virtual** — [Docs](https://tanstack.com/virtual/latest/docs/introduction)

**React** — [react.dev](https://react.dev) · [Suspense](https://react.dev/reference/react/Suspense) ·
[Hydration](https://react.dev/reference/react-dom/client/hydrateRoot)
**TypeScript** — [typescriptlang.org](https://www.typescriptlang.org/docs/)
**Vite** — [vite.dev](https://vite.dev/guide/) · [Environment API](https://vite.dev/guide/api-environment)
**Tailwind** — [tailwindcss.com](https://tailwindcss.com/docs/installation/using-vite) ·
**shadcn/ui** — [ui.shadcn.com](https://ui.shadcn.com/docs)
**PostgreSQL** — [docs](https://www.postgresql.org/docs/current/)
**Drizzle** — [orm.drizzle.team](https://orm.drizzle.team/docs/overview) ·
[drizzle-kit](https://orm.drizzle.team/docs/kit-overview)
**Prisma** — [prisma.io/docs](https://www.prisma.io/docs)
**Better Auth** — [docs](https://www.better-auth.com/docs) ·
[TanStack Start integration](https://www.better-auth.com/docs/integrations/tanstack)
**Testing** — [Vitest](https://vitest.dev) · [Testing Library](https://testing-library.com) ·
[MSW](https://mswjs.io) · [Playwright](https://playwright.dev)
**Deployment** — [Cloudflare Workers + TanStack Start](https://developers.cloudflare.com/workers/framework-guides/web-apps/tanstack-start/) ·
[Netlify TanStack Start guide](https://docs.netlify.com/build/frameworks/framework-setup-guides/tanstack-start/) ·
[Nitro](https://nitro.build) · [srvx](https://srvx.unjs.io)

## 31.5 Exercises

- **Beginner:** Read the official Start Overview + Comparison pages; rewrite this module's verdict table in your own words.
- **Intermediate:** Enable RSC in a *scratch* app; return a renderable from a server function; document what breaks and why it's labeled experimental.
- **Production:** Write the one-page “Why Start for Meridian” decision doc (for your future CTO), including risks and mitigations.
- **Architecture Challenge:** Same team, two apps: a marketing/content site and an internal ops SaaS. Assign frameworks to apps and justify with this module's tables.

🧠 **MENTAL MODEL — Comparisons:** frameworks are bundles of tradeoffs, not scores. Pick by *where your risk
lives*: type-safety/deployment control (Start), RSC/ecosystem maturity (Next), or router-only (SPA).

---

**Final module: [Module 32 — Capstone PRD, Architecture Review & Debugging Gym →](32-capstone-prd-review.md)**
