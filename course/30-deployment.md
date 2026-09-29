# Module 30 — Deployment, Runtimes, CI/CD & Production Migrations

> Capstone Stage 19. Start's superpower: **the same app builds for many runtimes** — because the server is
> just a fetch handler. Verified against the official Hosting guide on 2026-09-28.

---

## 30.1 The four words: BUILD → OUTPUT → RUNTIME → HOST

```
BUILD     vite build (or rsbuild) — compiles client bundle + server bundle
OUTPUT    dist/client (static) + server entry (@tanstack/react-start/server-entry)
RUNTIME   Node ≥22.12 · Bun · Cloudflare Workers isolate · Nitro-hosted targets
HOST      Cloudflare · Netlify · Railway · Vercel · Docker/VPS · Appwrite Sites
```

Deployment stops being exotic when you see it this way: hosts differ in *how they invoke your fetch
handler* and *what the runtime allows*. Everything else is identical.

## 30.2 The server entry — the universal seam `[SERVER]`

FILE: `src/server.ts` (optional; Start ships a default when absent)

```ts
import handler, { createServerEntry } from '@tanstack/react-start/server-entry'
import { createStartHandler, defaultStreamHandler, defineHandlerCallback } from '@tanstack/react-start/server'

// 1) Typed request context for the whole middleware chain (module augmentation — verified pattern)
declare module '@tanstack/react-router' {
  interface Register {
    server: { requestContext: { requestId: string; buildId: string } }
  }
}

// 2) Custom handler hook: logging, error mapping, THEN the default streaming SSR
const startHandler = createStartHandler(defineHandlerCallback((ctx) => defaultStreamHandler(ctx)))

export default createServerEntry({
  async fetch(request, opts) {
    const requestId = request.headers.get('x-request-id') ?? crypto.randomUUID()
    return startHandler({
      request,
      ...opts,
      context: { requestId, buildId: process.env.BUILD_ID ?? 'dev' }, // typed everywhere downstream
    })
  },
})
```

This is where production bootstraps: DB pool bootstrap, logger setup, global middleware registration (via
`src/start.ts`), health hooks. On Cloudflare, the same file can also export Workers constructs (queues,
scheduled events, Durable Objects).

## 30.3 Deploy targets (official guides, verified patterns)

### Cloudflare Workers ⭐ official partner

```bash
npm i -D @cloudflare/vite-plugin wrangler
```

```ts
// vite.config.ts
import { cloudflare } from '@cloudflare/vite-plugin'
export default defineConfig({
  plugins: [
    cloudflare({ viteEnvironment: { name: 'ssr' } }),
    tanstackStart(),
    viteReact(),
  ],
})
```

```jsonc
// wrangler.jsonc
{
  "name": "meridian",
  "compatibility_date": "2025-09-02",
  "compatibility_flags": ["nodejs_compat"],
  "main": "@tanstack/react-start/server-entry"   // ← Start's default entry, hosted by Workers
}
```

`wrangler deploy` ships it. Note the implications (30.4): HTTP Postgres drivers, Workers KV/Queues for jobs.

### Netlify ⭐ official partner

```bash
npm i -D @netlify/vite-plugin-tanstack-start   # + plugin in vite.config (any position)
npx netlify deploy                              # build settings auto-detected for new projects
```

Manual alternative: `netlify.toml` with `command = "vite build"`, `publish = "dist/client"`.

### Railway ⭐ official partner — via the Nitro flow (see below), push-to-deploy.

### Nitro (multi-host: AWS Lambda, Azure, etc.)

Add Nitro v3 as the host layer (`nitro/vite` plugin) and pick its preset; the Start docs' hosting guide has
the current recipe — this is also how you reach targets without a bespoke Start integration.

### Node.js + Docker (self-hosted)

```dockerfile
# simplified; multi-stage in reality
FROM node:22-slim AS build
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:22-slim
WORKDIR /app
COPY --from=build /app/dist ./dist
ENV NODE_ENV=production PORT=3000
CMD ["node", "dist/server/index.mjs"]   # ← serve the built server bundle (check your Start version's output path)
```

(The official auth example runs the built server with `srvx`, a small production server from the unjs
family — `pnpx srvx --prod -s ../client dist/server/server.js`. Whatever serves it: bind `0.0.0.0`, set
health checks, put TLS termination in front.)

### Vercel / Bun / others

The hosting guide lists current recipes (Vercel via its build integration; Bun by running the server bundle
under Bun). Check the guide's section for your host — this area evolves fastest.

## 30.4 Runtime differences — the honest table

| Concern | Node ≥22 | Cloudflare Workers (edge) |
|---|---|---|
| Model | long-lived process(es) | isolate per request family, global edge |
| Filesystem | yes | no (KV/R2/D1/Durable Objects instead) |
| Node APIs | full | `nodejs_compat` subset |
| DB drivers | native TCP pools (`pg`) | HTTP/WebSocket drivers (Neon HTTP, Hyperdrive) |
| Connections | warm pool per instance | per-isolate; use pooled endpoints |
| Cold starts | rare (resident) | near-zero (isolates) but first-query latency exists |
| Jobs | Redis/BullMQ etc. | Queues + scheduled Workers |
| Secrets | host env/vault | `wrangler secret` |

Decision: **Node/Docker** = familiar ops + any driver; **Workers** = edge latency + managed ops, but audit
your dependency tree for Node-only assumptions *before* committing.

## 30.5 CI/CD pipeline (the production pipeline)

```
push
 ├─ lint (eslint/oxlint) + format check
 ├─ typecheck (tsc --noEmit)
 ├─ unit tests (vitest)
 ├─ integration tests (vitest + docker Postgres) — migrations run here first
 ├─ build (vite build)  ← catches bundling/import-protection errors
 ├─ E2E (playwright) against the built app
 └─ deploy
     ├─ migrations: apply PENDING, non-destructive migrations
     └─ swap traffic (host-managed or blue/green)
```

Rules: migrations run **automatically before** the new code serves; the pipeline never applies a
destructive migration (30.6); preview environments get seeded ephemeral DBs.

## 30.6 Migrations in production — zero-downtime discipline

**Expand → migrate → contract:**

1. **Expand:** add columns/tables (additive, safe) — deploy.
2. **Migrate data:** backfill in batches while both shapes work.
3. **Contract:** remove old column only after the previous release is fully rolled out and verified.

- Destructive changes (drop, rename, type change) ship as *multiple* non-destructive steps.
- Rollback plan = “previous code must run on the new schema” during the window.
- Long backfills: batched, resumable, throttled; never one `UPDATE everything` at peak.

## 30.7 Anti-patterns

⚠️ **DO NOT** read server env vars that only exist in one host's dashboard without boot-validation — fail
at deploy, not at first request.

⚠️ **DO NOT** assume Node APIs on edge (and vice versa) — keep runtime-specific code behind modules picked
at build time.

⚠️ **DO NOT** run migrations manually “when you remember” — the pipeline owns them.

⚠️ **DO NOT** deploy the dev command (`vite dev`) to production. It happens more than anyone admits.

## 30.8 Exercises

- **Beginner:** Dockerize Meridian (multi-stage), boot it, and hit `/api/health` from outside the container.
- **Intermediate:** Deploy to Cloudflare Workers with a Neon HTTP database; document every change you had to make (driver, secrets, jobs).
- **Production:** Full CI per 30.5 with preview deploys per PR; add a health-check + rollback runbook.
- **Architecture Challenge:** Plan the expand/migrate/contract sequence for: renaming `orders.status` values and splitting `users.name` into first/last — without downtime.
- **Debug Challenge:** “Works on Node deploy, 500s on Workers.” Enumerate runtime-caused suspects (TCP driver, `fs`, `process`, top-level await of network) and the minimal repro for each.

🧠 **MENTAL MODEL — Deployment:** your app is a *fetch handler with a build pipeline*. Hosts are sockets it
plugs into; runtimes are the laws of physics in that room. Pick the room whose physics your dependencies
already obey.

---

**Next: [Module 31 — Start vs Next.js vs React Router · Experimental Features · Docs Map →](31-comparisons-experimental-docs-map.md)**
