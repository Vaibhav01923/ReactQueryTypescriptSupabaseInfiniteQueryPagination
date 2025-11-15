# Infinite Pagination with Supabase & React Query

A single reference you can return to anytime. This guide walks you step‑by‑step from creating `utils/paginate.ts` to writing a robust Supabase fetch function, integrating with `useInfiniteQuery`, adding an IntersectionObserver-based auto‑loader, and advanced patterns (debounced search, filters, reset behavior).

---

## Table of contents

1. Overview & mental model
2. Files you will create
3. `utils/paginate.ts` (page-number pagination)
4. Supabase function: `functions/supabaseFunctions.ts` (typed, with `count`)
5. Optional: `hooks/useAlbums.ts` (custom hook)
6. `components/LoadMoreTrigger.tsx` (IntersectionObserver)
7. `app/page.tsx` (integrate everything)
8. Variants & advanced topics

   * Button vs auto-load
   * Debounced search + infinite scroll
   * Sorting & filters
   * Resetting pagination on filter/search change
9. Troubleshooting & common pitfalls
10. Best practices & performance tips

---

## 1 — Overview & mental model

* There are two common pagination models: **page-number** and **offset**.
* This guide uses **page-number** (0, 1, 2...) because your `getRange(page, limit)` uses `page * limit`.
* `pageParam` (React Query) will represent a **page number** in our app.
* Each page fetch returns `{ data, currentPage, nextPage }`. React Query uses `getNextPageParam` to know which pageParam to pass next.
* To avoid an unnecessary final fetch, use Supabase's `count: "exact"` so you can compute whether a next page exists before requesting it.

---

## 2 — Files you will create

```
src/
  functions/
    supabaseFunctions.ts   // the server-side fetch function
  hooks/
    useAlbums.ts           // optional custom hook
  components/
    LoadMoreTrigger.tsx    // IntersectionObserver trigger
  app/
    page.tsx               // the page using useInfiniteQuery
  utils/
    paginate.ts            // getRange helper
```

---

## 3 — `utils/paginate.ts` (page-number pagination)

```ts
// src/utils/paginate.ts

// returns a range of numbers to be used for pagination.
export function getRange(page: number, limit: number) {
  const from = page * limit;
  const to = from + limit - 1;

  return [from, to];
}
```

**Why this pattern?**

* `page = 0` → `from = 0`, `to = limit-1` (first page)
* `page = 1` → `from = limit`, `to = (2*limit)-1` (second page)
* This keeps `pageParam` as a page number (easier mental model) and converts to ranges needed by Supabase.

---

## 4 — Supabase function: `functions/supabaseFunctions.ts`

This function is typed, uses `count: "exact"` to know the total number of rows, and returns `nextPage` as a page number or `null`.

```ts
// src/functions/supabaseFunctions.ts
import { supabase } from "@/supabase-client";
import { Tables } from "../../database.types"; // adjust path to your types
import { getRange } from "@/utils/paginate";

type Album = Tables<"albums">;

export async function getAlbums(
  { pageParam }: { pageParam: number }
): Promise<{
  data: Album[];
  currentPage: number;
  nextPage: number | null;
}> {
  const LIMIT = 50;
  const [from, to] = getRange(pageParam, LIMIT);

  const { data, error, count } = await supabase
    .from("albums")
    .select("*", { count: "exact" })
    .range(from, to);

  if (error || !data) {
    console.error("Error fetching albums:", error);
    return {
      data: [],
      currentPage: pageParam,
      nextPage: null,
    };
  }

  const total = count ?? 0;
  const nextStart = (pageParam + 1) * LIMIT;
  const nextPage = nextStart >= total ? null : pageParam + 1;

  return {
    data,
    currentPage: pageParam,
    nextPage,
  };
}
```

**Key points:**

* `count` is the total rows in the table (when `select(..., { count: 'exact' })` is used).
* `nextPage` is computed using the total to avoid the wasted final fetch.
* Typed return helps TypeScript + React Query interop.

---

## 5 — Optional: `hooks/useAlbums.ts` (custom hook)

Creating a custom hook cleans the UI code and centralizes caching keys and options.

```ts
// src/hooks/useAlbums.ts
"use client";

import { useInfiniteQuery } from "@tanstack/react-query";
import { getAlbums } from "@/functions/supabaseFunctions";

export function useAlbums() {
  return useInfiniteQuery({
    queryKey: ["albums"],
    queryFn: ({ pageParam = 0 }) => getAlbums({ pageParam }),
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextPage,
  });
}
```

**Why a hook?**

* Reuse in multiple components
* Keep UI components focused on rendering
* Easier to add options like `staleTime`, `cacheTime`, `refetchOnWindowFocus` in one place

---

## 6 — `components/LoadMoreTrigger.tsx` (IntersectionObserver)

This tiny client component observes a tiny div at the bottom of the list and calls `onVisible()` when it becomes visible.

```tsx
// src/components/LoadMoreTrigger.tsx
"use client";

import { useEffect, useRef } from "react";

export function LoadMoreTrigger({ onVisible }: { onVisible: () => void }) {
  const ref = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    const el = ref.current;
    if (!el) return;

    const observer = new IntersectionObserver((entries) => {
      if (entries[0].isIntersecting) {
        onVisible();
      }
    });

    observer.observe(el);

    return () => observer.disconnect();
  }, [onVisible]);

  return <div ref={ref} style={{ height: 1 }} />;
}
```

**Notes**

* `use client` is required because `IntersectionObserver` and refs are browser APIs.
* `height: 1` makes it tiny but observable. You can tweak styling or add margins to control prefetch distance.

---

## 7 — `app/page.tsx` (integrate everything)

You can either use `useInfiniteQuery` directly in the page or call the custom hook.

```tsx
// src/app/page.tsx
"use client";

import { LoadMoreTrigger } from "@/components/LoadMoreTrigger";
import { useAlbums } from "@/hooks/useAlbums"; // or use useInfiniteQuery directly

export default function Home() {
  const {
    data,
    isLoading,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useAlbums();

  if (isLoading) return <div>Loading...</div>;

  const albums = data?.pages.flatMap((page) => page.data) ?? [];

  return (
    <div>
      {albums.map((album) => (
        <div key={album.id}>{album.title}</div>
      ))}

      {hasNextPage && (
        <LoadMoreTrigger
          onVisible={() => {
            if (!isFetchingNextPage) fetchNextPage();
          }}
        />
      )}

      {isFetchingNextPage && <div>Loading more...</div>}
    </div>
  );
}
```

**Notes**

* `data.pages` is an array of page objects returned by your `getAlbums` function.
* Use `flatMap` to flatten the `pages -> data[]` into a single list.

---

## 8 — Variants & advanced topics

### Button vs auto-load

* Button (Load more) is simpler and gives users control. Use `fetchNextPage()` on button click.
* Auto-load (IntersectionObserver) is more natural for feeds. Use a small sentinel div and call `fetchNextPage()` when visible.

### Debounced Search + Infinite Scroll

* Keep your search value in client state.
* Debounce the input (e.g., `useDebounce` or lodash `debounce`) before passing the search to the query key.
* When search changes, **reset the query** and fetch page 0.

Pattern:

```ts
const [query, setQuery] = useState("");
const debounced = useDebounce(query, 300);

useInfiniteQuery({
  queryKey: ["albums", { q: debounced }],
  queryFn: ({ pageParam = 0 }) => getAlbums({ pageParam, q: debounced }),
  getNextPageParam: (last) => last.nextPage,
});
```

Make sure `getAlbums` accepts the filter and applies it to the Supabase query (e.g., `.ilike('title', `%${q}%`)`).

### Sorting & Filters

* Include current filters & sort options in the `queryKey`. Example:
  `['albums', { q, sortBy, genre }]`
* Passing different `queryKey` values will cause React Query to treat it as a new query (good).

### Resetting pagination on filter change

* Either use the `queryKey` approach (recommended) or call `queryClient.resetQueries(['albums'])` & `fetchNextPage({ pageParam: 0 })`.
* Best approach: when the `queryKey` changes, React Query will fetch fresh pages for the new key. Use `keepPreviousData: false` for a clean UI.

### Prefetching next page

* You can call `queryClient.prefetchInfiniteQuery(['albums', opts], ...)` with `pageParam` to warm the cache.

### SSR / Next.js considerations

* Server Components cannot hold client-only instances like QueryClient — wrap QueryProvider in a client component (e.g., `providers.tsx` with `"use client"`).
* For server-rendered first page, fetch page 0 server-side and hydrate the cache client-side.

---

## 9 — Troubleshooting & common pitfalls

* **`pageParam` confusion**: Be explicit whether it’s page number vs offset. Your `getRange` uses page number.
* **Missing `getNextPageParam`**: Without it, React Query won’t know how to fetch the next page.
* **Not returning `nextPage`** from your fetch function — React Query needs it.
* **Passing non-serializable props across RSC/Client boundary** (QueryClient, classes, functions) — make providers client components.
* **Empty page at the end**: Use `count: 'exact'` to prevent a wasted last fetch.

---

## 10 — Best practices & performance tips

* Use `LIMIT` sizes appropriate for your UI. Smaller pages = faster first paint, but more requests.
* Avoid large `select('*')` if you only need a few fields.
* Memoize item rendering (e.g., `React.memo` or virtualization) when rendering large lists.
* Consider `react-window` or `react-virtual` for very large lists to avoid DOM bloat.
* Add a small threshold or rootMargin to the observer to prefetch earlier (e.g., `rootMargin: '200px'`) so content is ready when user reaches it.
* Use `staleTime` and `cacheTime` in React Query to tune cache behavior.

---

### Final note

Keep this markdown file near your project. Copy-paste the code blocks into your repo and adapt types/paths. If you want, I can also:

* produce a `LoadMoreTrigger` debug build with console logs
* convert these examples to offset-based pagination instead
* show the debounced search implementation code and `getAlbums` filter example

Tell me which extra variant you'd like to add and I'll update the doc.
