# Module 19 — TanStack Form: Type-Safe Forms → Server Functions

> Capstone forms: login, product CRUD, org settings, checkout. We evaluate the ecosystem first — then go
> all-in on the TanStack way, because Form reached **stable v1** (`1.33.5`, verified) and pairs natively
> with Start's typed server boundary.

---

## 19.1 React Hook Form vs TanStack Form (2026)

| Criterion | React Hook Form | TanStack Form v1 |
|---|---|---|
| Maturity | Very mature, huge ecosystem | Stable v1 (graduated 2025), active |
| Type safety | Good (schema-driven) | **Exceptional** — field paths inferred from `defaultValues`; nested/array paths are literal types |
| Validation | Schema-first (resolvers) | Inline functions **or** Standard Schema (Zod 4 plugs in directly) |
| Framework reach | React-centric | Core is framework-agnostic (same core as Start's other libs) |
| Server integration | Manual | First-class with server functions (same-type inputs/outputs) |
| Ecosystem add-ons | Massive | Growing (Start is the primary consumer) |

**Course decision: TanStack Form.** RHF remains a fine choice in existing codebases — but in a Start app,
keeping the whole stack on TanStack type primitives pays off (one mental model, one devtools story).

## 19.2 Anatomy of a form `[CLIENT]`

FILE: `src/features/products/product-form.tsx`

```tsx
import { useForm } from '@tanstack/react-form'
import { ProductInputSchema, type ProductInput } from '~/server/schemas' // [BOTH] schema — one source
import { useServerFn } from '@tanstack/react-start'

export function ProductForm({ onDone }: { onDone: (p: Product) => void }) {
  const createProduct = useServerFn(createProductFn)

  const form = useForm({
    defaultValues: {
      name: '',
      description: '',
      priceCents: 0,
      categoryId: '',
    } satisfies ProductInput,

    // Zod 4 is a Standard Schema — pass it straight in (verify adapter details for your version)
    validators: { onSubmit: ProductInputSchema },

    onSubmit: async ({ value, formApi }) => {
      try {
        const product = await createProduct({ data: value })   // typed across the wire
        onDone(product)
      } catch (err) {
        if (isValidationError(err)) {
          // Server-side validation failed (defense in depth kicked in) — map onto fields
          for (const issue of err.issues) {
            formApi.setError(issue.path.join('.'), issue.message)
          }
        } else if (isForbidden(err)) {
          formApi.setError('', 'You do not have permission to create products.')
        } else {
          formApi.setError('', 'Something went wrong. Please try again.')
        }
      }
    },
  })

  return (
    <form
      onSubmit={(e) => { e.preventDefault(); form.handleSubmit() }}
      noValidate
      aria-describedby="product-form-errors"
    >
      <form.Field name="name">
        {(field) => (
          <div>
            <label htmlFor={field.name}>Name</label>
            <input
              id={field.name} name={field.name}
              value={field.state.value}
              onBlur={field.handleBlur}
              onChange={(e) => field.handleChange(e.target.value)}
              aria-invalid={field.state.meta.errors.length > 0}
              aria-describedby={`${field.name}-error`}
            />
            {field.state.meta.errors.map((err, i) => (
              <p key={i} id={`${field.name}-error`} role="alert" className="text-destructive text-sm">{err}</p>
            ))}
          </div>
        )}
      </form.Field>

      {/* price: integer cents, formatted as currency on display (Module 16 rule) */}
      <form.Field name="priceCents">
        {(field) => <MoneyInput field={field} />}
      </form.Field>

      {/* submit button knows form-wide state */}
      <form.Subscribe selector={(state) => [state.canSubmit, state.isSubmitting]}>
        {([canSubmit, isSubmitting]) => (
          <button type="submit" disabled={!canSubmit || isSubmitting}>
            {isSubmitting ? 'Saving…' : 'Create product'}
          </button>
        )}
      </form.Subscribe>
    </form>
  )
}
```

What you just got for free: typed field paths, per-field + form-level errors, `canSubmit`/`isSubmitting`
state, client validation before the wire, and server validation mapped back onto the same fields.

## 19.3 Async validation (server-backed) `[CLIENT → SERVER]`

```tsx
<form.Field
  name="slug"
  validators={{
    onChangeAsyncDebounceMs: 400,
    onChangeAsync: async ({ value }) => {
      const taken = await checkSlugAvailable({ data: { slug: value } }) // GET server function
      return taken ? undefined : 'This slug is already in use'
    },
  }}
>
```

Rules: debounce; cancel stale checks (Form handles races by field identity); never treat “available” as a
server guarantee — re-check inside `createProduct` (race window).

## 19.4 Dynamic fields & arrays `[CLIENT]`

```tsx
// Order line items: typed array paths like `items[0].quantity`
<form.Field name="items">
  {(fieldArray) => (
    <>
      {fieldArray.state.value.map((_, i) => (
        <div key={i} className="flex gap-2">
          <form.Field name={`items[${i}].quantity`}>
            {(qty) => <NumberInput field={qty} min={1} max={99} />}
          </form.Field>
          <button type="button" onClick={() => fieldArray.removeValue(i)}>Remove</button>
        </div>
      ))}
      <button type="button" onClick={() => fieldArray.pushValue({ productId: '', quantity: 1 })}>
        Add item
      </button>
    </>
  )}
</form.Field>
```

## 19.5 Server-error mapping — the contract

| Server error | Form behavior |
|---|---|
| 422 validation (`issues[]`) | Set per-field errors via path |
| 401 | Global error + redirect to `/login?redirect=…` |
| 403 | Global error “not allowed” (UI hid a button the user reached anyway — investigate!) |
| 409 conflict | Global or field error (“slug taken”, “version outdated — reload”) |
| 5xx/network | Global retry-able error; keep field values intact |

Never clear the user's work on error. Ever.

## 19.6 Accessibility checklist for every form

- Every input: associated `<label htmlFor>` (or `aria-label`).
- Errors: `role="alert"` + `aria-invalid` + `aria-describedby` chain (screen readers announce).
- Focus management: on submit error, move focus to the first invalid field.
- Disable-vs-busy: prefer `aria-busy` + preventing double-submit over disabling inputs (disabled controls
  drop out of tab order and confuse AT).
- Native input types (`type="email"`, `inputMode="numeric"`) before JS cleverness.

## 19.7 Anti-patterns

⚠️ **DO NOT** skip client validation “because the server validates anyway” — UX dies; the server check is
security, the client check is respect for the user.

⚠️ **DO NOT** build controlled state outside the form for form-owned fields (two stores = one bug).

⚠️ **DO NOT** alert() errors; map them into the form per 19.5.

## 19.8 Exercises

- **Beginner:** Build the login form from Module 14 with TanStack Form + Zod; server 401 → global error, fields preserved.
- **Intermediate:** Product form with async slug check and the currency field (cents ↔ display).
- **Production:** Checkout: dynamic items array, server repricing on submit, 409 handling when stock changes mid-checkout (recompute + field-level “price changed” notice).
- **Architecture Challenge:** Which validations belong client-only (e.g. “confirm password matches”), both, or server-only (e.g. org quota)? Make the table.
- **Debug Challenge:** Users report occasional duplicate orders on slow connections. Diagnose the double-submit path and fix with idempotency (Module 12) + form state — not by disabling the button alone.

🧠 **MENTAL MODEL — Forms:** the form is a *typed, validated conversation*: field → schema (instant) →
server function `.validator` (authoritative) → error issues back onto fields. One schema, three stops.

---

**Next: [Module 20 — TanStack Table: Admin-Grade Data Grids →](20-tables.md)**
