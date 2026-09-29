# Module 21 — TanStack Virtual: Rendering Ten Thousand Things

> Verified current: `@tanstack/react-virtual@3.14.13`. Virtualization is a performance tool with real UX
> and accessibility costs — this module teaches when to reach for it and when to walk past it.

---

## 21.1 Normal rendering vs virtualization

| | Normal list | Virtualized list |
|---|---|---|
| DOM nodes | one per item | one per **visible** item (+ overscan) |
| 10k items | ~10k nodes → slow paint, huge layout cost, jank | ~30 nodes → flat cost |
| Scroll | browser-native, trivially correct | math: offsets, measurement, transform |
| Find-in-page, print, a11y | free | requires care (21.5) |
| Rule | default | only when measured jank exists or list is unbounded |

**Decision rule:** virtualize when (a) the list is effectively unbounded (activity feed with infinite
scroll), or (b) you *measured* frame drops above a few hundred heavy rows. Not before.

## 21.2 The virtualizer — core pattern `[CLIENT]`

FILE: `src/features/activity/activity-feed.tsx`

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'
import { useRef } from 'react'

export function ActivityFeed({ items }: { items: Activity[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 56,          // estimated row height (px)
    overscan: 8,                     // render 8 extra rows above/below → fewer blank flashes
  })

  return (
    <div
      ref={parentRef}
      role="list"
      aria-label="Activity feed"
      className="h-[600px] overflow-auto"
    >
      {/* Total-size spacer creates the real scrollbar length */}
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map((vi) => {
          const item = items[vi.index]
          return (
            <div
              key={item.id}
              role="listitem"
              data-index={vi.index}
              ref={virtualizer.measureElement}   // ⬅ measures REAL height (variable rows!)
              style={{
                position: 'absolute',
                top: 0,
                transform: `translateY(${vi.start}px)`,
                width: '100%',
              }}
            >
              <ActivityRow item={item} />
            </div>
          )
        })}
      </div>
    </div>
  )
}
```

Concepts that do the work:

- **`estimateSize`** — initial guess; wrong guesses just cause mild scroll jitter until measured.
- **`measureElement`** — real measurement per row → **variable-height rows for free**. Always use it unless
  every row is pixel-identical (then pass a fixed size and skip measurement for speed).
- **`overscan`** — the buffer that hides virtualization from fast flicks.
- **Absolute positioning + translateY** — cheap for the compositor; no layout thrash.

## 21.3 Virtual + infinite scroll (Query partnership) `[CLIENT]`

```tsx
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: ['activity'],
  queryFn: ({ pageParam }) => getActivity({ data: { cursor: pageParam } }),
  initialPageParam: undefined as string | undefined,
  getNextPageParam: (last) => last.nextCursor,
})
const items = useMemo(() => data?.pages.flatMap((p) => p.items) ?? [], [data])

// Fetch the next page when the user scrolls near the end:
useEffect(() => {
  const last = virtualizer.getVirtualItems().at(-1)
  if (hasNextPage && !isFetchingNextPage && last && last.index >= items.length - 10) {
    fetchNextPage()
  }
}, [virtualizer.getVirtualItems(), items.length, hasNextPage, isFetchingNextPage, fetchNextPage])
```

Capstone use: the **activity feed** streams in via Module 09's widgets on first paint, then pages infinitely
on scroll — virtualized so a week of activity costs the same as ten rows.

## 21.4 Virtualizing a TABLE (large product list) `[CLIENT]`

Two honest options:

1. **Server pagination (Module 20)** — preferred for admin tables: smaller pages, simpler a11y, URL state.
2. **Virtualized body** — when you genuinely need one continuous scrollable grid: render the `<thead>`
   normally, virtualize only `<tbody>` rows with the same spacer technique, and keep column widths
   synchronized (`table-layout: fixed` + identical column templates).

Don't virtualize what pagination solves better.

## 21.5 Accessibility & UX obligations (non-negotiable)

- **Semantic structure survives:** `role="list"/"listitem"` or real `<table>` rows; announce counts with
  `aria-rowcount`/`aria-setsize` + `aria-posinset` so screen readers convey “item 412 of 9,300”.
- **Keyboard:** focused row must scroll into view — call `virtualizer.scrollToIndex(index, { align: 'start' })`
  on focus changes. Items unmount off-screen; manage focus deliberately.
- **Find-in-page & print** don't see unrendered items — offer “load all” for print styles or accept it with
  a visible note.
- **Scroll anchoring** after prepends (new activity on top): preserve `virtualizer.scrollToOffset` or pin
  the user's place, or the feed jumps under their cursor.

## 21.6 Performance discipline

```
MEASURE → IDENTIFY → OPTIMIZE → MEASURE AGAIN
```

- React DevTools Profiler + Performance panel: is the cost *rendering rows* (virtualize) or *data
  processing* (fix the query)?
- Row components must be cheap: memoize heavy cells; don't create functions/objects per row render.
- `overscan` is the only knob you should tune by feel; everything else is measured.

## 21.7 Anti-patterns

⚠️ **DO NOT** virtualize lists under ~200 rows — you bought complexity and a11y debt for zero gain.

⚠️ **DO NOT** forget the total-size spacer — the scrollbar lies and jumps around.

⚠️ **DO NOT** combine `estimateSize` with wildly wrong guesses and then blame the library for “blank rows”.

⚠️ **DO NOT** use virtualization as a substitute for not loading 50k rows of data you never show (page it).

## 21.8 Exercises

- **Beginner:** Virtualize a 10,000-row fixed-height list; compare FPS in the Performance panel vs un-virtualized.
- **Intermediate:** Variable-height activity feed with `measureElement`; verify no blank flashes on fast flicks (tune overscan).
- **Production:** Infinite activity feed (21.3) with prepend-anchoring when new items arrive while the user is scrolled down.
- **Architecture Challenge:** Same feed needed in a print view and by keyboard-only users — design the degraded modes.
- **Debug Challenge:** “Scrollbar thumb jumps when scrolling up.” Which two parameters interact (estimateSize vs measured heights), and what's the fix?

🧠 **MENTAL MODEL — Virtualization:** render what's visible + a little more; the scroll container is *full
size*, the DOM is *window-sized*. It's a rendering optimization with UX contracts attached — pay them.

---

**Next: [Module 22 — Integration Chapter: Everything Cooperates (+ Optimistic UI) →](22-integration-optimistic.md)**
