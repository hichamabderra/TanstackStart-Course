# Module 00 — CURRENT VERIFIED TANSTACK START STACK (September 2026)

> This module is the course's contract with reality. Every version was pulled from the **npm registry
> `latest` dist-tag on 2026-09-28**; every status claim from the **official TanStack documentation** fetched
> the same day. If you're reading this later, re-run the checks at the bottom — the methodology matters more
> than the numbers.

---

## 0.1 The verified stack table

| Layer | Package / Tool | Verified version | Maturity | Official docs |
|---|---|---|---|---|
| Full-stack framework | `@tanstack/react-start` | **1.168.59** | 🟡 **Release Candidate** (feature-complete, API stable) | [start/latest/docs](https://tanstack.com/start/latest/docs/framework/react/overview) |
| Routing core | `@tanstack/react-router` | **1.170.40** | ✅ Stable v1 | [router/latest/docs](https://tanstack.com/router/latest/docs/framework/react/overview) |
| UI library | `react` / `react-dom` | **19.3.0** | ✅ Stable | [react.dev](https://react.dev) |
| Language | `typescript` | **7.0.2** (native compiler, GA) — 6.x fully supported too | ✅ Stable | [typescriptlang.org](https://www.typescriptlang.org) |
| Build tool | `vite` | **8.3.1** (Start peer: `>=7.0.0`; **Rsbuild `^2`** also officially supported) | ✅ Stable | [vite.dev](https://vite.dev) |
| Async state | `@tanstack/react-query` | **5.104.0** | ✅ Stable v5 | [query/latest/docs](https://tanstack.com/query/latest/docs/framework/react/overview) |
| Forms | `@tanstack/react-form` | **1.33.5** | ✅ **Stable v1** (Form graduated to stable) | [form/latest/docs](https://tanstack.com/form/latest/docs/framework/react/overview) |
| Tables | `@tanstack/react-table` | **9.2.4** | ✅ Stable **v9** (major rewrite; ships a `./legacy` export) | [table/latest/docs](https://tanstack.com/table/latest/docs/introduction) |
| Virtualization | `@tanstack/react-virtual` | **3.14.13** | ✅ Stable v3 | [virtual/latest/docs](https://tanstack.com/virtual/latest/docs/introduction) |
| Validation | `zod` | **4.6.5** (Zod 4 is the standard import) | ✅ Stable v4 | [zod.dev](https://zod.dev) |
| ORM (primary) | `drizzle-orm` | **0.45.3** | ✅ Stable (0.x versioning, production-used) | [orm.drizzle.team](https://orm.drizzle.team) |
| ORM (alternative) | `prisma` | **7.x** — the **official `start-basic-auth` example uses Prisma 7** with driver adapters | ✅ Stable | [prisma.io/docs](https://www.prisma.io/docs) |
| Database | PostgreSQL | 16 / 17 | ✅ | [postgresql.org](https://www.postgresql.org/docs/) |
| Authentication | `better-auth` | **1.7.6** — ships a first-class `better-auth/tanstack-start` integration | ✅ Stable | [better-auth.com/docs/integrations/tanstack](https://www.better-auth.com/docs/integrations/tanstack) |
| Styling | `tailwindcss` | **4.3.3** (CSS-first config, `@tailwindcss/vite` plugin) | ✅ Stable v4 | [tailwindcss.com](https://tailwindcss.com) |
| UI components | shadcn/ui | current (Radix UI primitives + Tailwind) | ✅ | [ui.shadcn.com](https://ui.shadcn.com) |
| Animation | `motion` | **13.4.4** (formerly Framer Motion) | ✅ Stable | [motion.dev](https://motion.dev) |
| Unit/integration testing | `vitest` | **5.0.2** | ✅ Stable v5 | [vitest.dev](https://vitest.dev) |
| Component testing | `@testing-library/react` | latest (v16.x line) | ✅ | [testing-library.com](https://testing-library.com) |
| API mocking | `msw` | **3.0.0** (v3 is current) | ✅ Stable v3 | [mswjs.io](https://mswjs.io) |
| E2E testing | `@playwright/test` | **1.63.0** | ✅ | [playwright.dev](https://playwright.dev) |
| Runtime/hosting | Nitro (optional host), `srvx`, Cloudflare Workers, Node ≥ **22.12** | — | ✅ | module 30 |

### Runtime requirement to note

`@tanstack/react-start` declares `engines: node >= 22.12.0`. Vite 8 requires `^20.19.0 || >=22.12.0`.
**Course policy: Node 22 LTS or Node 24.**

## 0.2 What is TanStack Start's maturity RIGHT NOW?

Straight from the official docs overview (fetched 2026-09-28):

> **“TanStack Start is currently in the Release Candidate stage! This means it is considered feature-complete
> and its API is considered stable. This does not mean it is bug-free or without issues…”**

Interpretation, as a senior engineer would give it:

1. **RC ≠ unstable API.** The API surface is what v1 will ship. Breaking changes between now and 1.0 are
   expected to be minor and documented.
2. **It is production-usable** — official hosting partners (Cloudflare, Netlify, Railway) ship first-party
   deployment integrations, and the TanStack team maintains production examples (`start-basic-auth`,
   `start-trellaux`, Clerk/Supabase/WorkOS examples).
3. **Pin your versions** (`1.168.x`), read release notes on upgrade, and keep an eye on the RC→1.0 notes.
4. The v1 RC was announced **September 23, 2025**; the same blog URL will be updated when 1.0 ships:
   [tanstack.com/blog/announcing-tanstack-start-v1](https://tanstack.com/blog/announcing-tanstack-start-v1).

### Feature-by-feature stability

| Feature | Status | Evidence |
|---|---|---|
| File-based + type-safe routing | ✅ Stable (Router v1) | Router is a separate, long-stable v1 library |
| Loaders / `beforeLoad` / route context / cache | ✅ Stable | Router core |
| Full-document SSR | ✅ Stable | Docs overview lists it as a core capability |
| Streaming SSR (Suspense-driven) | ✅ Stable | Overview: “full-document SSR, streaming” |
| **Server Functions** (`createServerFn`) | ✅ Stable | Dedicated guide, CSRF middleware, serialization checks |
| **Server Routes** (`server.handlers`) | ✅ Stable | Dedicated guide; unified with file routes |
| **Middleware** (request + server-function) | ✅ Stable | Dedicated guide, `createMiddleware` |
| Cookies utilities / sessions | ✅ Stable | `@tanstack/react-start/server` cookie helpers |
| Deployment integrations | ✅ Stable | Cloudflare (`@cloudflare/vite-plugin`), Netlify (`@netlify/vite-plugin-tanstack-start`), Railway (via Nitro), plus Nitro/Vercel/Node/Bun guides |
| **React Server Components** | ⚠️ **EXPERIMENTAL** | Docs: “Server Components are experimental! The API may see refinements.” Backed by `@tanstack/react-start-rsc` at version `0.1.x`. Opt-in: `tanstackStart({ rsc: { enabled: true } })` |

**Course rule:** the main curriculum is built 100% on stable APIs. RSC gets one clearly-labeled module (31)
explaining what it is, how it works, and when *not* to use it in production.

## 0.3 Recent history you must know (things old tutorials get wrong)

This is the part most content online gets silently wrong. Verify against the guides linked in each module.

### 🔁 OLD → MODERN #1: The build plugin

```ts
// 🔁 OLD (2024 / early 2025 — vinxi era, then the old import)
// import { defineConfig } from '@tanstack/react-start/config'   // gone
// import { tanstackStart } from '@tanstack/react-start/plugin-vite' // old path

// ✅ MODERN (Sept 2026): plain Vite (or Rsbuild) + the plugin subpath export
import { defineConfig } from 'vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    tanstackStart(),
    viteReact(), // ⚠️ the React plugin must come AFTER the Start plugin
  ],
})
```

### 🔁 OLD → MODERN #2: Server routes

```ts
// 🔁 OLD: createServerFileRoute('/api/hello', { methods: { GET: ... } })

// ✅ MODERN: server routes live INSIDE the same file-based route system
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/hello')({
  server: {
    handlers: {
      GET: async ({ request }) => new Response('Hello, World!'),
    },
  },
})
```

### 🔁 OLD → MODERN #3: Server function input validation

```ts
// 🔁 OLD: .inputValidator(...)
// ✅ MODERN: .validator(...)
const createUser = createServerFn({ method: 'POST' })
  .validator(UserSchema)
  .handler(async ({ data }) => { /* data: typed + validated */ })
```

### 🔁 OLD → MODERN #4: Deployment is no longer “Nitro-inside, by default”

Start's build is now plain Vite/Rsbuild with **deployment handled by host integrations**:

- **Cloudflare Workers** → `@cloudflare/vite-plugin` + `wrangler.jsonc` with `main: "@tanstack/react-start/server-entry"`
- **Netlify** → `@netlify/vite-plugin-tanstack-start`
- **Railway** → the documented Nitro flow
- **Nitro v3** → still available as a general host (`nitro/vite`) for targets like AWS Lambda, Azure, etc.
- **Node** → build then serve the server bundle (official examples use `srvx`)

### 🔁 OLD → MODERN #5: CSRF protection for server functions

Start now provides `createCsrfMiddleware()` and **auto-installs it for server functions** unless you define a
custom `src/start.ts` start instance — in which case you register it explicitly. Module 25 digs in.

### 🔁 OLD → MODERN #6: Root route head/script helpers

```tsx
// ✅ MODERN: from '@tanstack/react-router' directly
import { createRootRoute, Outlet, HeadContent, Scripts } from '@tanstack/react-router'

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { title: 'Meridian' },
    ],
  }),
  component: RootComponent, // renders <html><head><HeadContent/></head><body><Outlet/><Scripts/></body></html>
})
```

## 0.4 What I could NOT fully verify (honesty section)

Per the course's own rules, here is what is *not* verified against a primary source today:

1. **Exact guide page slugs** for some sub-topics (e.g. a standalone “streaming” or “database” guide page) —
   the Start docs reorganize pages frequently; streaming and DB guidance are covered inside other guides.
   Where I couldn't fetch a page, I say so in the module.
2. **Table v9's full API delta** vs v8. The `@tanstack/react-table@9.2.4` package is verified, and it ships a
   `./legacy` entry, but module 20 flags every spot where you should double-check v9 docs against our snippets.
3. **TanStack Start + Zod 4 adapter details** — `.validator()` accepts Zod 4 schemas (the official guide shows
   exactly this), but edge cases (e.g. error formatting) should be tested in your app.

Everything else in the course is backed by pages fetched on 2026-09-28, linked inline.

## 0.5 Why each technology {#why-each-technology}

| Pick | Why (and what beat the alternative) |
|---|---|
| **Vite** over Rsbuild | Both are first-class in Start; the official examples, all partner plugins, and the Cloudflare/Netlify integrations lead with Vite. We mention Rsbuild where relevant. |
| **Drizzle** as primary ORM | SQL-first (you learn the actual queries), no codegen build step, tiny runtime, first-class edge/Workers support. **Prisma 7 is the documented alternative** — it's what the *official* Start auth example uses, and its DX/schema tooling is excellent; on Cloudflare Workers you use its driver-adapter mode. Module 16 has the full comparison. |
| **Better Auth** | Framework-agnostic auth with an **official TanStack Start integration** (`tanstackStartCookies`), sessions in the DB (revocable, no JWT-only pitfalls), plugins for email-OTP, 2FA, organization/multi-tenant. The alternative (hand-rolled sessions) is *taught anyway* in module 14, because you must understand what the library does. |
| **Zod 4** | De-facto standard; used by TanStack's own guides for server-function validation and `validateSearch`. |
| **Tailwind v4 + shadcn/ui** | Tailwind 4 is the current standard (CSS-first config, Vite plugin); shadcn gives accessible Radix primitives you own and can modify — not a black-box component library. |
| **Vitest + Playwright + MSW** | Vitest shares Vite config with Start (zero-config module resolution), Playwright for real-browser E2E, MSW for API-boundary mocks. All three are at new majors (5 / 1.63 / 3) — modules 28 uses current APIs. |
| **Motion** | The maintained successor name of Framer Motion; used for view/page/list transitions with reduced-motion support. CSS View Transitions are covered as the native alternative. |

## 0.6 How to re-verify this yourself (docs-first learning, step 0)

```bash
# 1) Versions straight from the registry (source of truth for "latest")
npm view @tanstack/react-start version
npm view @tanstack/react-router version
npm view @tanstack/react-query version
npm view @tanstack/react-form version
npm view @tanstack/react-table version
npm view @tanstack/react-virtual version

# 2) The official docs (ALWAYS the /latest/ namespace for current)
open https://tanstack.com/start/latest/docs/framework/react/overview

# 3) Maturity statement + feature list
#    → the “Note” box at the top of the Overview page

# 4) Every docs page has a plain-markdown twin — perfect for reading/LLMs:
curl -s https://tanstack.com/start/latest/docs/framework/react/overview.md | head -40

# 5) Release line & blog
open https://tanstack.com/blog/announcing-tanstack-start-v1
```

## 0.7 Module 00 mental model

🧠 **MENTAL MODEL — “Verify, then build.”**
Frameworks move faster than tutorials. Before learning any API: (1) read the *official* overview page,
(2) check the npm `latest` version and engine constraints, (3) note the maturity label, (4) search the docs
for the specific API name — if the docs don't show it, assume your source is outdated. This habit is worth
more than any single module in this course.

---

**Next: [Module 01 — The TanStack Start Mental Model & Execution Model →](01-mental-model.md)**
