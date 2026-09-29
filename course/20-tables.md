# Module 20 — TanStack Table: Admin-Grade Data Grids

> Capstone admin screens (users, products, orders) are tables. TanStack Table is **headless**: it computes
> state (sorting, selection, pagination) and renders nothing — you own the markup, Tailwind/shadcn own the
> pixels. Verified current: `@tanstack/react-table@9.2.4` (v9; it ships a `./legacy` export — double-check
> any helper names against the v9 docs as you build).

---

## 20.1 Two operating modes — decide first

| Mode | Who does the work | When |
|---|---|---|
| **Client-side** (`getSortedRowModel` etc.) | Table sorts/paginates loaded rows in memory | Small datasets (< ~1–2k rows), already-fetched data |
| **Server-driven** (`manualSorting`, `manualPagination`, `manualFiltering`) | Your API + URL state (Modules 05/16) | Real admin tables: 10k+ rows, tenant scoping, indexes |

Meridian's admin tables are **server-driven**; the table state is *projected into the URL* so it's shareable
and SSR-correct.

## 20.2 Column definitions `[BOTH]` (definitions are data)

FILE: `src/features/admin/users/columns.tsx`

```tsx
import type { ColumnDef } from '@tanstack/react-table'
import type { AdminUserRow } from '~/types/entities'

export const userColumns: ColumnDef<AdminUserRow>[] = [
  {
    id: 'select',
    header: ({ table }) => (
      <Checkbox
        aria-label="Select all"
        checked={table.getIsAllPageRowsSelected()}
        onCheckedChange={(v) => table.toggleAllPageRowsSelected(!!v)}
      />
    ),
    cell: ({ row }) => (
      <Checkbox
        aria-label={`Select ${row.original.email}`}
        checked={row.getIsSelected()}
        onCheckedChange={(v) => row.toggleSelected(!!v)}
      />
    ),
    enableSorting: false,
  },
  { accessorKey: 'name', header: 'Name', cell: ({ getValue }) => <span className="font-medium">{getValue<string>()}</span> },
  { accessorKey: 'email', header: 'Email' },
  { accessorKey: 'role', header: 'Role', cell: ({ getValue }) => <RoleBadge role={getValue<Role>()} /> },
  {
    accessorKey: 'createdAt',
    header: 'Joined',
    cell: ({ getValue }) => <time dateTime={getValue<string>()}>{formatDate(getValue<string>())}</time>,
    // server-driven: we map table sorting state → API `sort` param ourselves
  },
  { id: 'actions', cell: ({ row }) => <UserRowActions user={row.original} /> },
]
```

## 20.3 The table instance + server-driven wiring `[CLIENT]`

FILE: `src/features/admin/users/users-table.tsx`

```tsx
import { useReactTable, getCoreRowModel, flexRender } from '@tanstack/react-table'
import { useNavigate, useSearch } from '@tanstack/react-router'

export function UsersTable() {
  // 1) State lives in the URL (Module 05) — the table is a projection of it
  const { page, pageSize, sort, dir, q } = useSearch({ from: '/_admin/admin.users' })
  const navigate = useNavigate()

  // 2) Data comes from Query (Module 18) keyed by that same state
  const { data, isFetching } = useQuery({
    queryKey: adminUserKeys.list({ page, pageSize, sort, dir, q }),
    queryFn: () => adminListUsers({ data: { page, pageSize, sort, dir, q } }),
    placeholderData: (prev) => prev,   // keep rows visible while the next page loads
  })

  // 3) The table computes nothing server-side — it renders + reports intent
  const table = useReactTable({
    data: data?.rows ?? [],
    columns: userColumns,
    rowCount: data?.total ?? 0,
    getCoreRowModel: getCoreRowModel(),
    manualSorting: true,
    manualPagination: true,
    manualFiltering: true,
    state: {
      sorting: [{ id: sort, desc: dir === 'desc' }],
      pagination: { pageIndex: page - 1, pageSize },
    },
    onSortingChange: (updater) => {
      const next = typeof updater === 'function' ? updater(table.getState().sorting) : updater
      const s = next[0]
      navigate({ search: (p) => ({ ...p, sort: s?.id ?? 'createdAt', dir: s?.desc ? 'desc' : 'asc', page: 1 }), replace: true })
    },
    onPaginationChange: (updater) => {
      const next = typeof updater === 'function' ? updater(table.getState().pagination) : updater
      navigate({ search: (p) => ({ ...p, page: next.pageIndex + 1, pageSize: next.pageSize }), replace: true })
    },
  })

  return (
    <div className="rounded-md border">
      <table className="w-full text-sm" aria-busy={isFetching}>
        <thead>
          {table.getHeaderGroups().map((hg) => (
            <tr key={hg.id}>
              {hg.headers.map((h) => (
                <th key={h.id} aria-sort={
                  h.column.getIsSorted() ? (h.column.getIsSorted() === 'desc' ? 'descending' : 'ascending') : 'none'
                }>
                  {h.isPlaceholder ? null : (
                    h.column.getCanSort() ? (
                      <button onClick={h.column.getToggleSortingHandler()} className="flex items-center gap-1">
                        {flexRender(h.column.columnDef.header, h.getContext())}
                        {h.column.getIsSorted() === 'asc' ? '↑' : h.column.getIsSorted() === 'desc' ? '↓' : '↕'}
                      </button>
                    ) : flexRender(h.column.columnDef.header, h.getContext())
                  )}
                </th>
              ))}
            </tr>
          ))}
        </thead>
        <tbody>
          {table.getRowModel().rows.map((row) => (
            <tr key={row.id} data-state={row.getIsSelected() && 'selected'}>
              {row.getVisibleCells().map((cell) => (
                <td key={cell.id}>{flexRender(cell.column.columnDef.cell, cell.getContext())}</td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
      <TablePager table={table} total={data?.total ?? 0} />
    </div>
  )
}
```

The loop you must see: **URL → Query → table state → user intent → URL**. The table never owns truth; it
renders it and reports intent. Refresh, deep-link, back button — all correct by construction.

## 20.4 Feature checklist (how each is done)

| Feature | Mechanism |
|---|---|
| Sorting | `onSortingChange` → URL `sort/dir` (server-driven) |
| Filtering | Toolbar inputs → URL `q`/`category` (Module 05) — NOT the table's filter state, for server mode |
| Pagination | `manualPagination` + `rowCount`; page size selector writes URL |
| Row selection | `getIsSelected`/`toggleSelected`; bulk actions read `table.getSelectedRowModel().rows` |
| Column visibility | `columnVisibility` state → dropdown writing it (persist to user prefs if worth it) |
| Empty/loading/error | 4-state house style (Module 24): skeletons rows while `isFetching`, `<EmptyState>` when `total === 0` |
| Accessibility | `<table>` semantics, `aria-sort` on headers, labeled selection checkboxes |

### Sidebar: when `LIMIT/OFFSET` breaks down

Past ~100k rows or deep pages, offset pagination degrades (and “page 412” is meaningless for live data).
Switch to **keyset/cursor pagination**: URL carries `cursor` instead of `page`; Query becomes
`useInfiniteQuery`; “Next” replaces page numbers. Table v9 + infinite queries compose cleanly.

## 20.5 Anti-patterns

⚠️ **DO NOT** let the table own filter state that's also in the URL — two stores, one bug (Module 05's rule).

⚠️ **DO NOT** client-sort a server-paginated table (you'd sort one page of many — silently wrong).

⚠️ **DO NOT** render 10k rows “because it's fine” — Module 21 exists for large in-memory lists.

⚠️ **DO NOT** forget `placeholderData` — page changes flash to skeleton and feel broken.

## 20.6 Exercises

- **Beginner:** Build the admin products table (client-side mode, 20 seeded rows) with sorting + selection.
- **Intermediate:** Convert to server-driven with URL-synced sort/page/filter against your Module 16 repository.
- **Production:** Add bulk actions (archive N products in one server function + undo toast) with optimistic row state.
- **Architecture Challenge:** Design the orders table for ORG_OWNER vs ADMIN: same component, different columns/permissions — where do capabilities come from (Module 15)?
- **Debug Challenge:** “Sorting by Joined works but pagination jumps to page 1 silently.” Trace the `onSortingChange` handler: what must resetting `page` do, and why did a teammate's version forget it?

🧠 **MENTAL MODEL — Tables:** TanStack Table is a **state machine for tabular intent**; the URL is the store,
Query is the fetcher, the server is the sorter. The `<table>` is just the view at the end of the pipeline.

---

**Next: [Module 21 — TanStack Virtual: Lists at Scale →](21-virtual.md)**
