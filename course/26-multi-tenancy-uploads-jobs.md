# Module 26 — Multi-Tenancy, File Uploads & Background Jobs

> Capstone Stage 11 + 15. Three features that separate demo apps from SaaS: tenants that can't see each
> other, files that are safe and served correctly, and work that doesn't block the request.

---

## 26.1 Multi-tenancy — the data model

```
Organization A                    Organization B
├── memberships (users+roles)     ├── memberships
├── products                      ├── products
└── orders                        └── orders
        │                                 │
        └──────── ONE DATABASE ───────────┘
                 (shared schema, orgId on every tenant row)
```

FILE: `src/server/db/schema.ts` (from Module 16) — the load-bearing decisions:

- Every tenant-owned table carries `orgId NOT NULL` with FK + index.
- `memberships` links users → orgs with `orgRole` (`OWNER`/`MEMBER`); platform `users.role` (`USER`/`ADMIN`)
  is separate. Better Auth's `organization()` plugin can own this surface if you prefer library-managed
  memberships — pick ONE owner for membership logic and stick to it.
- User's **active org** travels in the session (selected at login or switched in UI), never inferred from a
  client-supplied parameter.

### Isolation — enforced at the data boundary (Module 15 recap, now mandatory)

```ts
// [SERVER] — repositories make the tenant UNFORGETTABLE: no list function compiles without orgId
async listForOrg(orgId: string, opts) { … where(eq(t.orgId, orgId)) … }

// Cross-tenant test suite (Module 25.4): for each resource endpoint,
// assert user-from-org-A gets 404/403 on org-B's resource.
```

> ⚠️ **DO NOT** implement isolation as “a middleware that sets a global currentOrg and queries read it
> implicitly” — implicit ambient state is how tenants leak. Explicit `orgId` arguments, enforced by types.

### Tenant resolution in requests

```ts
// [SERVER] — middleware or service helper: derive org from SESSION, validate membership
export async function resolveActiveOrg(session: Session) {
  const orgId = session.activeOrganizationId ?? (await firstMembership(session.user.id))
  const membership = await membershipsRepo.find(session.user.id, orgId)
  if (!membership) throw Forbidden('Not a member of this organization')
  return { orgId, orgRole: membership.orgRole }
}
```

Org switching = re-authenticate the session context (Better Auth: `setActive`), then invalidate Query +
router caches — user B's org A data must never paint for org B.

## 26.2 File uploads — production style

### The flow

```
Browser ──multipart/form-data──▶ Server Function (POST, FormData) [SERVER]
   → validate (size, type, magic bytes) → stream to OBJECT STORAGE
   → store { key, ownerId, orgId, contentType, size, visibility } in Postgres
   → serve via signed URLs (private) or CDN URL (public assets)
```

FILE: `src/server/uploads.functions.ts` `[SERVER]`

```ts
export const uploadProductImage = createServerFn({ method: 'POST' })
  .validator((data) => {
    if (!(data instanceof FormData)) throw new Error('Expected FormData')
    return data   // server functions accept FormData on POST (verified)
  })
  .handler(async ({ data }) => {
    const session = await ensureSession()
    const file = data.get('image')
    if (!(file instanceof File)) throw Unauthorized()

    // 1) size limit BEFORE reading the stream (also enforce at proxy/host level)
    if (file.size > 5 * 1024 * 1024) throw new AppError(413, 'FILE_TOO_LARGE', 'Max 5 MB')

    // 2) type: check declared type AND magic bytes (declare is trivially spoofed)
    const kind = await detectKind(file)                 // sniff first bytes: jpeg/png/webp only
    if (!['image/jpeg', 'image/png', 'image/webp'].includes(kind))
      throw new AppError(422, 'BAD_FILE_TYPE', 'Images only (jpeg/png/webp)')

    // 3) store under tenant-scoped, unguessable keys — NEVER user-supplied filenames
    const key = `orgs/${session.orgId}/products/${crypto.randomUUID()}.${extFor(kind)}`
    await storage.put(key, file.stream(), { contentType: kind })

    return uploadsRepo.record({ key, orgId: session.orgId, ownerId: session.user.id, kind, size: file.size })
  })
```

### Why object storage and not the app filesystem

- Server instances are disposable and horizontally scaled — a disk write on instance A is invisible to B.
- Edge/Workers runtimes may have **no** persistent filesystem at all (Module 30).
- Storage services give you durability, lifecycle rules, CDNs, and signed URLs for free.

### Access control for files

- **Public assets** (catalog images): serve via CDN with `content-disposition: inline`, immutable keys.
- **Private files** (invoices): never public; mint **short-lived signed URLs** per request after re-checking
  `canView(user, resource)`. The URL is capability + expiry; the DB row is truth.
- Set `content-disposition: attachment` for anything executable-ish; never store/upload HTML that gets
  served same-origin (stored XSS).

## 26.3 Background jobs — work that shouldn't hold the request

| Job | Trigger | Pattern |
|---|---|---|
| Send email (verification, receipts) | after signup/order | enqueue → worker renders + sends |
| Image processing (thumbnails) | after upload | enqueue with storage key |
| Reports/analytics rollups | schedule (nightly) | cron/scheduled trigger |
| Notifications fan-out | on events | enqueue per recipient batch |
| Cleanup (expired sessions, orphan uploads) | schedule | cron |

**Conceptual architecture (same shape on every runtime):**

```
Server Function ──enqueue(payload)──▶ Queue (durable) ──▶ Worker(s) ──▶ side effect
        │                                                     │
        └── returns “accepted” immediately                    └── idempotent by job key, retries w/ backoff
```

Runtime choices (Module 30 pairs these with deployments):

- **Cloudflare Workers:** Queues + scheduled Workers (native, no extra infra).
- **Node hosts:** Redis + BullMQ (or Postgres-backed queues at small scale).
- **Tiny scale:** `waitUntil`-style fire-and-forget for emails *only if* losing one is acceptable — usually
  it isn't; prefer a real queue.

Job rules: **idempotent** (dedupe key), **small payloads** (ids, not blobs), **retryable** (side effects
guarded), **observable** (Module 29 logs job start/end/failure).

## 26.4 Anti-patterns

⚠️ **DO NOT** trust `Content-Type` or file extensions — sniff magic bytes; re-encode images when paranoid.

⚠️ **DO NOT** store user-supplied filenames as keys (path traversal, collisions).

⚠️ **DO NOT** do image resizing inside the upload request — that's a job.

⚠️ **DO NOT** rely on “middleware sets currentOrg” ambient state (26.1).

## 26.5 Exercises

- **Beginner:** Implement org switching in the session; prove Query/router caches clear on switch.
- **Intermediate:** Ship the product-image upload with all validations; store locally in dev (MinIO), S3-compatible in staging.
- **Production:** Signed-URL download for invoices with per-request authz; exploit test: attempt cross-org signed URL minting.
- **Architecture Challenge:** Notification fan-out to 50k members: design queue topology, batching, backpressure, and the “mark all read” query.
- **Debug Challenge:** “Occasional 500s on uploads only in production behind a proxy.” Suspect list: body-size limits at proxy, streaming not supported, timeouts — verify each with the right probe.

🧠 **MENTAL MODEL — Tenancy & assets:** the tenant id rides in the *session*, appears in *every query*, and
is tested as an *attack*. Files are *capabilities* (keys + signed URLs), not paths. Jobs are *deferred
contracts* — durable, idempotent, observable.

---

**Next: [Module 27 — Performance Engineering →](27-performance.md)**
