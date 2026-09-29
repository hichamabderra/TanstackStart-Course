# Module 23 — UI System: Tailwind v4, shadcn/ui, Motion & Accessibility

> Capstone Stage 3. The goal is a small, owned design system — not a component-library dependency you can't
> audit. Everything here assumes Tailwind **4.3.x** and shadcn on Radix primitives (versions from Module 00).

---

## 23.1 Tailwind v4 as the design-token layer

Tailwind v4 is **CSS-first**: tokens live in CSS, not a JS config file (Module 02 set the plugin up).

```css
/* src/styles.css [BOTH] */
@import 'tailwindcss';

@theme {
  --color-brand: oklch(0.55 0.18 255);
  --color-surface: oklch(0.99 0 0);
  --radius-card: 0.75rem;
  --font-sans: 'Inter', ui-sans-serif, system-ui;
}

@custom-variant dark (&:where(.dark, .dark *));
```

Rules:

- **Tokens are the contract** (`--color-brand`), utilities are the usage (`bg-brand`). Changing a token
  restyles the app; chasing utility names across 400 files is how design systems die.
- **Dark mode** = `class="dark"` on `<html>` + `dark:` variants. Persist choice server-side (cookie) so SSR
  paints the right theme — no flash (Module 08's hydration lesson).
- **Responsive first**: design mobile → `md:`/`lg:` up. Admin tables get horizontal scroll containers on
  small screens, not squeezed columns.

## 23.2 shadcn/ui — components you own

```bash
npx shadcn@latest add button card dialog dropdown-menu command table form input select
npx shadcn@latest add skeleton toast sheet tooltip avatar badge
```

Why shadcn over a packaged library:

1. Source is copied into `src/components/ui/` → you can **audit, adapt, and tree-shake** (Module 25 audits
   what it ships).
2. Built on **Radix primitives** → dialog focus-trap, menu keyboard nav, `aria-*` wiring are engineered,
   not approximated.
3. Tailwind-native → matches 23.1's token system via CSS variables.

**Component coverage for Meridian** (where each comes from):

| Need | Component |
|---|---|
| Modals (edit order, confirm delete) | `Dialog` (focus trap, Esc, overlay click) |
| Row actions | `DropdownMenu` |
| ⌘K navigation | `Command` (command menu) |
| Forms | `Form` wrappers wired to TanStack Form fields (Module 19) |
| Tables | `Table` primitives + Module 20 logic |
| Side panels (mobile filters) | `Sheet` |
| Feedback | `toast` (sonner-based) |
| Loading | `Skeleton` — every data surface (Module 06/24) |
| Empty | Hand-rolled `<EmptyState icon title action>` — ship one, reuse everywhere |

## 23.3 The four visual states — a house style

Every data surface renders exactly one of: **Loading** (skeleton, layout-matched) → **Empty** (friendly,
actionable CTA) → **Error** (what happened + retry) → **Success**. Modules 06/24 wire the logic; this module
owns the look. Reuse `<StateSwitch state={...}>` so no screen improvises.

## 23.4 Animation — Motion for React (`motion@13.4.4`)

```tsx
// [CLIENT] — dialog/list transitions
import { motion, AnimatePresence } from 'motion/react'

function OrderRows({ rows }: { rows: AdminOrderRow[] }) {
  return (
    <AnimatePresence initial={false}>
      {rows.map((row) => (
        <motion.tr
          key={row.id}
          layout                          // smooth reorder on sort
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
          transition={{ duration: 0.15 }}
        >
          …
        </motion.tr>
      ))}
    </AnimatePresence>
  )
}
```

Where animation earns its keep in an ops app:

- **Layout animations** on table sort/filter (`layout` prop) — communicates *what changed*.
- **Dialog/sheet mount/unmount** — prevents jarring pops.
- **Optimistic transitions** (Module 22): heart scale-pop on favorite = perceptible confirmation.
- **Page transitions**: subtle fade/slide via route-level wrappers — or the **native View Transitions API**
  (React 19.x has built-in View Transition support — check react.dev for the current component name and
  Start integration status before adopting; native = cheapest).

### Reduced motion — always

```css
/* [BOTH] */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

Motion honors this if you enable `MotionConfig reducedMotion="user"` — do it at the root.

## 23.5 Accessibility — part of every feature, not a module you visit

Checklist enforced across the course (and audited in Module 32):

| Area | Requirement |
|---|---|
| Semantics | Real `<button>`, `<table>`, `<nav>`, headings in order; no div-onclick |
| Keyboard | Every interactive reachable & operable; visible focus ring; logical tab order |
| Focus management | Dialog: trap on open, restore on close; route change: move focus to `<h1>` |
| Forms | Labels, `aria-invalid`, `aria-describedby` error chain (Module 19) |
| Tables | `aria-sort`, labeled selection checkboxes (Module 20) |
| Virtual lists | `aria-rowcount/posinset`, keyboard scroll-into-view (Module 21) |
| Live updates | `aria-live="polite"` for toasts/notification badge |
| Color | WCAG AA contrast; never color alone (badges carry text/icons) |
| Motion | Reduced-motion respected (23.4) |
| Images | Meaningful alt text; decorative = `alt=""` |

Tooling: axe-core in CI (via Playwright `@axe-core/playwright`), Lighthouse in PRs, keyboard-only manual pass
per feature.

## 23.6 Anti-patterns

⚠️ **DO NOT** install a closed component library "to go faster" then fight its theming and a11y for months.

⚠️ **DO NOT** animate by default — animate *meaning* (state changes), not decoration.

⚠️ **DO NOT** build dark mode with a parallel stylesheet; one token set, two values.

⚠️ **DO NOT** treat skeletons as optional — layout shift without them looks broken.

## 23.7 Exercises

- **Beginner:** Define Meridian's token set (brand, surface, radius, font) + dark mode toggle persisted in a cookie.
- **Intermediate:** Build `<StateSwitch>` + `<EmptyState>` + skeleton components; apply to `/products`.
- **Production:** ⌘K command menu (Command component) navigating typed routes — keyboard-only usable.
- **Architecture Challenge:** Where should the theme live: cookie (SSR-correct), localStorage, or media-query-only? Decide and justify against the hydration-flash constraint.
- **Debug Challenge:** Screen readers announce dialog content twice / focus escapes the modal. Inspect Radix wiring vs a hand-rolled portal — what primitive did the teammate re-implement badly?

🧠 **MENTAL MODEL — UI system:** tokens define the language, primitives guarantee behavior, four states make
it predictable, motion explains change, accessibility makes it universal. All owned, all auditable.

---

**Next: [Module 24 — Error Handling: Every Layer, 401 vs 403, Four States →](24-error-handling.md)**
