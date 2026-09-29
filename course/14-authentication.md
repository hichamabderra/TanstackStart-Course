# Module 14 — Authentication: Sessions, Cookies, Better Auth

> Capstone Stage 5. Authentication is where most tutorials wave their hands. We won't: you'll understand the
> full cookie/session machinery, then use **Better Auth** — which ships an *official TanStack Start
> integration* (verified 2026-09-28) — and know exactly what it's doing for you.

---

## 14.1 The complete flow (memorize this diagram)

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant SF as Login Server Function
  participant A as Auth core (Better Auth)
  participant DB as Postgres (sessions table)
  B->>SF: POST credentials (email, password)
  SF->>A: signInEmail
  A->>DB: verify password hash, create session row
  A-->>B: Set-Cookie: session_token=… (HttpOnly, Secure, SameSite=Lax)
  Note over B: cookie stored; JS can NEVER read it
  B->>SF: later request — cookie sent automatically
  SF->>A: getSession(headers)
  A->>DB: session row exists? expired? revoked?
  A-->>SF: { user, session } → route context
  SF-->>B: personalized response
  Note over B,DB: logout = delete session row + expire cookie (revocation!)
```

## 14.2 Sessions & cookies — the mechanics

**Authentication = WHO ARE YOU?** Proven once at login, then *remembered* via a session.

The session token cookie, configured correctly:

| Flag | Value | Why |
|---|---|---|
| `HttpOnly` | ✅ always | JavaScript cannot read it → XSS can't steal the session |
| `Secure` | ✅ in production | only sent over HTTPS |
| `SameSite` | `Lax` (default) | blocks cross-site POSTing of the cookie → CSRF mitigation; `Strict` where you can afford it |
| `Path` | `/` | sent app-wide |
| `Max-Age` | days–weeks | plus server-side expiry on the session row |
| `Domain` | exact host | never wider than needed (multi-tenant caveat, Module 26) |

**Revocation** is the reason sessions live in the database: logout/password-change/“sign out everywhere”
delete or bump the row; the next `getSession` fails. JWT-only schemes can't do this without extra machinery
— a real production difference.

**Password hashing:** never roll your own; a serious auth layer uses Argon2id/scrypt/bcrypt with sane
costs. Better Auth handles this; the lesson is that **plaintext never touches logs or errors** either.

## 14.3 Better Auth setup (verified integration)

FILE: `src/lib/auth.ts` `[SERVER]`

```ts
import { betterAuth } from 'better-auth'
import { drizzleAdapter } from 'better-auth/adapters/drizzle'
import { tanstackStartCookies } from 'better-auth/tanstack-start' // ← official Start integration
import { organization } from 'better-auth/plugins'                 // multi-tenancy (Module 26)
import { emailOTP } from 'better-auth/plugins'                     // email verification / OTP
import { twoFactor } from 'better-auth/plugins'                    // MFA
import { db } from '~/server/db/index.server'

export const auth = betterAuth({
  database: drizzleAdapter(db, { provider: 'pg' }),
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,          // server-only env (Module 29)
  emailAndPassword: { enabled: true, requireEmailVerification: true },
  session: { cookieCache: { enabled: true, maxAge: 60 } },
  plugins: [
    organization(),
    emailOTP({ async sendVerificationOTP({ email, otp }) { /* queue email — Module 26 */ } }),
    twoFactor(),
    tanstackStartCookies(),  // ⚠️ MUST be LAST: bridges Set-Cookie into Start's response handling
  ],
})
```

### Mount the HTTP handler — a SERVER ROUTE (Module 11)

FILE: `src/routes/api/auth/$.ts` `[SERVER]`

```ts
import { createFileRoute } from '@tanstack/react-router'
import { auth } from '~/lib/auth'

export const Route = createFileRoute('/api/auth/$')({
  server: {
    handlers: {
      GET: async ({ request }) => auth.handler(request),
      POST: async ({ request }) => auth.handler(request),
    },
  },
})
```

### Session server functions (the official pattern)

FILE: `src/server/auth/auth.functions.ts`

```ts
import { createServerFn } from '@tanstack/react-start'
import { getRequestHeaders } from '@tanstack/react-start/server'
import { auth } from '~/lib/auth'

export const getSession = createServerFn({ method: 'GET' }).handler(async () => {
  const headers = getRequestHeaders()                 // cookies arrive via request headers
  return await auth.api.getSession({ headers })       // null if unauthenticated
})

export const ensureSession = createServerFn({ method: 'GET' }).handler(async () => {
  const headers = getRequestHeaders()
  const session = await auth.api.getSession({ headers })
  if (!session) throw new Error('Unauthorized')       // Module 24 maps this to 401
  return session
})
```

## 14.4 Protecting routes (UX layer) and functions (security layer)

**Routes — `beforeLoad` + pathless layout** (verified pattern):

FILE: `src/routes/_authed.tsx` `[BOTH]`

```tsx
import { createFileRoute, redirect, Outlet } from '@tanstack/react-router'
import { getSession } from '~/server/auth/auth.functions'

export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    const session = await getSession()
    if (!session) throw redirect({ to: '/login', search: { redirect: location.href } })
    return { user: session.user }        // typed context for all children (Module 06)
  },
  component: () => <Outlet />,
})
```

**Server functions — the REAL boundary:**

```ts
export const createOrder = createServerFn({ method: 'POST' })
  .validator(OrderInput)
  .handler(async ({ data }) => {
    const session = await ensureSession()   // ← every function re-proves identity
    return ordersService.create(data, session.user) // ownership enforced inside (Module 15)
  })
```

> ⚠️ **DO NOT** rely on `_authed.beforeLoad` for security. It's UX. Every function/route authorizes itself.

## 14.5 Login / Logout / Register (UI wiring)

FILE: `src/routes/login.tsx` `[CLIENT form → SERVER]`

```tsx
import { createFileRoute } from '@tanstack/react-router'
import { useServerFn } from '@tanstack/react-start'
import { authClient } from '~/lib/auth-client'   // better-auth client SDK (createAuthClient)

export const Route = createFileRoute('/login')({ component: LoginPage })

function LoginPage() {
  return (
    <LoginForm
      onSubmit={async ({ email, password, redirect }) => {
        const { error } = await authClient.signIn.email({ email, password })
        if (error) return { formError: error.message }
        window.location.href = redirect ?? '/dashboard' // full reload → fresh session everywhere
      }}
    />
  )
}
```

```ts
// [CLIENT] — logout (cookie cleared server-side; drop any client caches too)
await authClient.signOut()
queryClient.clear()                 // Module 18: no leaking previous user's cache
await router.invalidate()
```

The TanStack Form version of `LoginForm` (typed errors, pending state) is Module 19.

## 14.6 The rest of production auth (concept map)

| Concern | How we handle it | Module |
|---|---|---|
| Password reset | Better Auth flow + email job | here + 26 |
| Email verification | `requireEmailVerification` + `emailOTP` | here |
| MFA | `twoFactor` plugin (TOTP) | here |
| Rate limiting login | per-IP/per-account limiter middleware before auth DB hits | 25 |
| Session expiration | DB row expiry + `getSession` check | here |
| Revocation | delete session rows (“sign out everywhere”) | here |
| Concurrent sessions list | sessions table query → settings page | 26 |
| Account lockout / suspicious-login alerts | policy layer on top | 26 |

## 14.7 Anti-patterns

⚠️ **DO NOT** store session tokens in `localStorage`/React state — XSS-readable. HttpOnly cookies only.

⚠️ **DO NOT** put user objects in long-lived client caches without clearing on login/logout.

⚠️ **DO NOT** trust `beforeLoad` (again). Or client-side role checks. Or “the button was hidden”.

⚠️ **DO NOT** log tokens, passwords, or full cookies. Redact headers in request logs (Module 29).

⚠️ **DO NOT** skip `Secure` on cookies in production “because it works locally”.

## 14.8 Exercises

- **Beginner:** Wire Better Auth with email/password; build login/register/logout; verify the cookie flags in DevTools (HttpOnly? SameSite?).
- **Intermediate:** Implement `_authed` with redirect-back; add an “already logged in → redirect to /dashboard” guard on `/login` (watch out for loops).
- **Production:** Add email verification (OTP via a logged “email” in dev), password reset, and session revocation on password change; write E2E tests for each.
- **Architecture Challenge:** Multi-device sessions: design the settings UI + data model (list, revoke one, revoke all). What happens to an open tab when its session is revoked? (Hint: next server round-trip 401s — design the client recovery, Module 24.)
- **Debug Challenge:** “Login works on localhost but cookies vanish on the preview deployment.” Enumerate causes using 14.2's table (Secure flag on http, domain mismatch, SameSite + cross-host preview, proxy stripping Set-Cookie) and the exact check for each.

🧠 **MENTAL MODEL — Authentication:** prove identity once, then ride a bearer token inside a locked
(HttpOnly) envelope. Every request re-asks the database “still valid?” — the cookie is convenience, the
session row is truth.

---

**Next: [Module 15 — Authorization & RBAC: Roles, Permissions, Ownership →](15-authorization-rbac.md)**
