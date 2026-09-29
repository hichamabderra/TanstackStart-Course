# Module 02 — Project Setup: Vite, Tailwind v4, and the Anatomy of a Start App

> Capstone Stage 1: **Setup.** By the end of this module you'll have the `meridian` app skeleton, and you'll
> understand the purpose of *every* file in it.

---

## 2.1 Three ways to start (verified 2026-09-28)

**Option A — TanStack Builder** (official recommended, AI-first setup flow):
👉 [tanstack.com/builder](https://tanstack.com/builder)

**Option B — The CLI** (what we use in this course):

```bash
npx @tanstack/cli@latest create
# prompts: project name, package manager, add-ons (Tailwind CSS, ESLint, Better Auth, …)
```

The CLI offers add-ons including **Tailwind CSS** and **Better Auth** — both course picks.

**Option C — From an official example** (great for diffing later):

```bash
npx gitpick TanStack/router/tree/main/examples/react/start-basic meridian
cd meridian && npm install && npm run dev
```

Official examples worth knowing: `start-basic`, `start-basic-react-query`, `start-basic-auth`,
`start-trellaux`, `start-clerk-basic`, `start-supabase-basic`, `start-workos`.

**Option D — From scratch** (module 2.3, because you must know what the CLI did for you).

> 🏭 **PRODUCTION PATTERN:** start with the CLI (Option B), then *read every generated file* against 2.3.
> Apps you can't explain are apps you can't debug.

## 2.2 Install matrix for the course stack

```bash
# Core (CLI gives you Start + Router + React + Vite + TS)
npm i @tanstack/react-start @tanstack/react-router react react-dom
npm i -D vite @vitejs/plugin-react typescript @types/react @types/react-dom @types/node

# Course additions
npm i @tanstack/react-query @tanstack/react-form @tanstack/react-table @tanstack/react-virtual zod
npm i drizzle-orm better-auth
npm i -D drizzle-kit @tailwindcss/vite tailwindcss
npm i -D vitest @playwright/test msw
```

## 2.3 Building from scratch — the five config files

### `package.json` `[META]`

```json
{
  "name": "meridian",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite dev",
    "build": "vite build",
    "start": "node .output/server/index.mjs",
    "test": "vitest",
    "test:e2e": "playwright test",
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate"
  }
}
```

(`"type": "module"` is required. The exact `start` command depends on your deployment target — Module 30.)

### `tsconfig.json` `[META]` — per official guidance

```json
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "moduleResolution": "Bundler",
    "module": "ESNext",
    "target": "ES2022",
    "skipLibCheck": true,
    "strict": true,
    "strictNullChecks": true,
    "paths": { "~/*": ["./src/*"] }
  }
}
```

> ⚠️ **DO NOT** enable `verbatimModuleSyntax`. The official docs warn it can cause **server bundles to leak
> into client bundles** — exactly the kind of secret-exfiltration bug Module 25 hunts for sport.

### `vite.config.ts` `[META]`

```ts
import { defineConfig } from 'vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'
import tailwindcss from '@tailwindcss/vite'
import viteReact from '@vitejs/plugin-react'

export default defineConfig({
  server: { port: 3000 },
  resolve: { tsconfigPaths: true },
  plugins: [
    tanstackStart(),
    tailwindcss(),
    viteReact(), // ⚠️ MUST come after the Start plugin (official requirement)
  ],
})
```

> 🔁 **OLD → MODERN:** Start used to be configured via `@tanstack/react-start/config` and later
> `plugin-vite`; the current import is the subpath export **`@tanstack/react-start/plugin/vite`**.
> Rsbuild users: `import { tanstackStart } from '@tanstack/react-start/plugin/rsbuild'`.

### `src/router.tsx` `[BOTH]`

```tsx
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export function getRouter() {
  return createRouter({
    routeTree,
    scrollRestoration: true,
    // Module 18 adds Query's dehydrate/hydrate here
  })
}

declare module '@tanstack/react-router' {
  interface Register {
    router: ReturnType<typeof getRouter>
  }
}
```

### `src/routes/__root.tsx` `[SERVER → CLIENT]`

```tsx
import type { ReactNode } from 'react'
import { Outlet, createRootRoute, HeadContent, Scripts } from '@tanstack/react-router'

export const Route = createRootRoute({
  head: () => ({
    meta: [
      { charSet: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { title: 'Meridian — Operations & Commerce' },
    ],
  }),
  component: RootComponent,
})

function RootComponent() {
  return (
    <RootDocument>
      <Outlet />
    </RootDocument>
  )
}

function RootDocument({ children }: Readonly<{ children: ReactNode }>) {
  return (
    <html suppressHydrationWarning>
      <head>
        <HeadContent />
      </head>
      <body className="bg-background text-foreground antialiased">
        {children}
        <Scripts />
      </body>
    </html>
  )
}
```

## 2.4 Anatomy of a Start project

```
meridian/
├── src/
│   ├── routes/
│   │   ├── __root.tsx          ← document shell, wraps everything
│   │   ├── index.tsx           ← "/"
│   │   └── …                   ← your route tree grows here (+ server routes!)
│   ├── router.tsx              ← createRouter + route tree registration
│   ├── routeTree.gen.ts        ← ⚙️ GENERATED. Never edit. Types come from here.
│   ├── start.ts                ← OPTIONAL: custom start instance (global request middleware)
│   └── server.ts               ← OPTIONAL: custom server entry (fetch handler)
├── vite.config.ts
├── tsconfig.json
└── package.json
```

Key facts:

- `routeTree.gen.ts` is regenerated whenever route files change. **Filesystem → generated tree → typed
  APIs → runtime navigation** is the pipeline Module 03 dissects.
- `src/start.ts` — when present, you own the middleware stack (incl. adding `createCsrfMiddleware()`
  explicitly). When absent, Start installs defaults (including CSRF protection for server functions).
- `src/server.ts` — custom server entry (Module 30): request context, logging, DB bootstrap.

## 2.5 Tailwind CSS v4 (CSS-first, no config file needed)

```css
/* src/styles.css [BOTH] */
@import 'tailwindcss';

@theme {
  /* design tokens live in CSS now — this IS the config */
  --color-brand: oklch(0.55 0.18 255);
  --font-sans: 'Inter', ui-sans-serif, system-ui, sans-serif;
}

@custom-variant dark (&:where(.dark, .dark *));
```

```tsx
// import it once from the root route
import '../styles.css'
```

> 🔁 **OLD → MODERN:** Tailwind v4 replaced `tailwind.config.js` JS config with **CSS-first `@theme`**, and
> the PostCSS plugin with the **Vite plugin** (`@tailwindcss/vite`). If a tutorial shows `content: [...]`
> globs, it's v3.

### shadcn/ui (add after Tailwind works)

```bash
npx shadcn@latest init     # pick your base color / CSS variables
npx shadcn@latest add button dialog dropdown-menu table input form skeleton toast
```

shadcn copies real component source into `src/components/ui/` — you own it, you can audit it (Module 25),
and it's built on **Radix primitives** (keyboard + a11y for free; Module 23). Verify current CLI flags at
[ui.shadcn.com](https://ui.shadcn.com) — the CLI moves fast.

## 2.6 Environment variables — the first security lesson

```bash
# .env — NEVER commit
# Server-only secrets (never prefixed):
DATABASE_URL=postgres://…
BETTER_AUTH_SECRET=…
# Public, ships to the browser (ONLY things safe for the browser):
VITE_APP_PUBLIC_URL=https://meridian.example.com
```

Rules you'll be drilled on in Module 29:

1. `VITE_*` = **build-time inlined into the client bundle**. Treat as public.
2. Everything else is server-process only — and even so, **anything a loader/component can read during SSR
   can leak through rendered HTML**. Secrets belong in server functions and `*.server.ts` modules.
3. Dangerous mistake preview: `const db = new Client(process.env.DATABASE_URL)` in a file imported by a
   component → the bundler may drag it client-side. Import protection (Module 01) exists to make this fail
   loudly.

## 2.7 Verify your setup

```bash
npm run dev
# → http://localhost:3000 renders __root.tsx + index.tsx
```

Checklist:

- [ ] Editing `src/routes/index.tsx` hot-reloads without full reload
- [ ] View Source on `/` shows **fully rendered HTML** (SSR is on) — not an empty `<div id="root">`
- [ ] Deleting/adding a route file regenerates `routeTree.gen.ts`
- [ ] `npm run build` produces `.output/` with `client/` and `server/` bundles

## 2.8 Exercises

- **Beginner:** Scaffold `meridian` via the CLI with Tailwind, then confirm the “View Source shows HTML” check.
- **Intermediate:** Re-create the project *from scratch* (2.3) without the CLI. Time yourself.
- **Production:** Add a `check` script: `tsc --noEmit && vitest run && playwright test`. Wire it to a git pre-push hook.
- **Architecture Challenge:** Which of these belong in `src/start.ts`, `src/server.ts`, or neither: request logging, DB connection bootstrap, global auth middleware, React Query client creation? (Answer at the end of Modules 13, 18, 30 — form your guess now.)

🧠 **MENTAL MODEL — Setup:** `vite.config.ts` decides **how** the app builds; `router.tsx` decides **what**
the app is; `routeTree.gen.ts` is the **typed bridge** between your files and the compiler; `start.ts`/`server.ts`
are the **optional seams** where production concerns (middleware, context, entry) plug in.

---

**Next: [Module 03 — TanStack Router Deep Dive: Route Trees & File-Based Routing →](03-router-deep-dive.md)**
