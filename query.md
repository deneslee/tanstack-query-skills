# TanStack Query

Query 5.104. Query is the only cache for server data.
Loaders prime it. Components observe it. Mutations invalidate it.

## Layers

```
lib/sharepoint        HTTP, OData, digest, field descriptors (no React)
features/*/api.ts     calls the lib, returns parsed domain objects
features/*/queries.ts keys, queryOptions, mutationOptions, small hooks
routes/*              loaders call queryClient.query(...)
components            useSuspenseQuery / useQuery / useMutation
```

Rules:
- `queryOptions` factories are the base abstraction. They work in loaders, components and imperative calls.
- Hooks are thin sugar on top. Never the only abstraction.
- Don't wrap with a wide `UseQueryOptions` param. Compose at the call site: `useQuery({ ...opts, select })`.
- Pass `signal` from the query function to `fetch`.

## Deprecated APIs (don't use)

| Deprecated | Use instead |
| --- | --- |
| `ensureQueryData(o)` | `query({ ...o, staleTime: 'static' })` |
| `fetchQuery(o)` | `query(o)` |
| `prefetchQuery(o)` | `query(o).catch(noop)` |
| `fetchInfiniteQuery` / `prefetchInfiniteQuery` / `ensureInfiniteQueryData` | `infiniteQuery(...)` (same patterns) |

`usePrefetchQuery` is fine.

## Keys and factories

One key root per feature. Invalidate by prefix.

```ts
// src/features/conferences/queries.ts
import { queryOptions, type QueryClient } from '@tanstack/react-query'
import { getConferences } from './api'

export const conferenceKeys = {
  all: ['conferences'] as const,
  items: () => [...conferenceKeys.all, 'items'] as const,
  // server mode later: list: (s: ConferenceFilters) => [...conferenceKeys.all, 'list', s] as const
}

export const conferenceQueries = {
  items: () =>
    queryOptions({
      queryKey: conferenceKeys.items(),
      queryFn: ({ signal }) => getConferences({ signal }),
      staleTime: 60_000,
    }),
}

// Used by route loaders
export const primeConferenceData = (qc: QueryClient) =>
  Promise.all([
    qc.query({ ...conferenceQueries.items(), staleTime: 'static' }),
    qc.query({ ...conferenceTagQueries.items(), staleTime: 'static' }),
    qc.query({ ...tagQueries.items(), staleTime: 'static' }),
    qc.query({ ...registrationQueries.items(), staleTime: 'static' }),
  ])
```

Freshness per data type:

| Data | staleTime |
| --- | --- |
| Conferences, ConferenceTags, Registrations, Follows | 60 s (default) |
| Tags, Categories, Organizations | 30 min |
| Current user | `'static'` |

No query-key-factory library. `queryOptions` already ties key, function and types together.

## QueryClient

```ts
// src/app/queryClient.ts
import { MutationCache, QueryCache, QueryClient, type QueryKey } from '@tanstack/react-query'
import { notifications } from '@mantine/notifications'
import { SpError } from '@/lib/sharepoint'

const isClientError = (e: unknown) => e instanceof SpError && e.status >= 400 && e.status < 500

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,
      retry: (failureCount, error) => !isClientError(error) && failureCount < 2,
    },
  },
  queryCache: new QueryCache({
    // First-load errors go to the route error component.
    // Only toast when old data is still on screen.
    onError: (error, query) => {
      if (query.state.data !== undefined) {
        notifications.show({ color: 'red', title: 'Refresh failed', message: error.message })
      }
    },
  }),
  mutationCache: new MutationCache({
    onSuccess: async (_data, _vars, _result, mutation, { client }) => {
      await invalidate(client, mutation.meta?.invalidates)
    },
    onError: (error, _vars, _result, mutation, { client }) => {
      void invalidate(client, mutation.meta?.invalidates) // part of the write may have happened
      notifications.show({ color: 'red', title: mutation.meta?.errorTitle ?? 'Save failed', message: error.message })
    },
  }),
})

function invalidate(client: QueryClient, keys?: ReadonlyArray<QueryKey>) {
  if (!keys?.length) return
  // 'all' also refetches queries that are not on screen right now
  return Promise.all(keys.map((queryKey) => client.invalidateQueries({ queryKey, refetchType: 'all' })))
}
```

Type the meta once:

```ts
// src/app/query-types.d.ts
import type { QueryKey } from '@tanstack/react-query'

declare module '@tanstack/react-query' {
  interface Register {
    mutationMeta: {
      invalidates?: ReadonlyArray<QueryKey>
      errorTitle?: string
    }
  }
}
```

Keep `defaultError` as `Error`. Zod and network errors are not `SpError`. Check with `instanceof`.

## Browser-side list boundary

All lists load whole. Join and filter happen in memory.
Pages, home sections, calendar and spotlight use the same two hooks.

```ts
// src/features/conferences/queries.ts
import { useSuspenseQueries } from '@tanstack/react-query'
import { joinConferences, queryConferences } from './derive'

export function useJoinedConferences() {
  const [conferences, links, tags, registrations] = useSuspenseQueries({
    queries: [conferenceQueries.items(), conferenceTagQueries.items(), tagQueries.items(), registrationQueries.items()],
  })
  return useMemo(
    () => joinConferences(conferences.data, links.data, tags.data, registrations.data),
    [conferences.data, links.data, tags.data, registrations.data],
  )
}

// The boundary. Server mode later only changes the inside of this hook.
export function useConferenceList(filters: ConferenceFilters): ListResult<ConferenceView> {
  const all = useJoinedConferences()
  return useMemo(() => queryConferences(all, filters), [all, filters])
}

export function useConferenceView(id: number) {
  return useJoinedConferences().find((c) => c.id === id) // undefined = just deleted
}
```

```ts
// src/lib/types.ts
export type ListResult<T> = { items: T[]; total?: number; nextCursor?: string }
```

`total` exists in browser mode. Server mode gives `nextCursor` (SharePoint has no `$skip` and no filtered totals).

Why a hook and not a "derived query" (a query whose function reads other queries)?
That query races with invalidation. It can read the old cached list or get its inner fetch cancelled.

`derive.ts` holds pure functions. That's where tests go:

```ts
// src/features/conferences/derive.ts
const norm = (s: string) => s.normalize('NFD').replace(/\p{Diacritic}/gu, '').toLowerCase()

// date-only strings compare correctly as strings
const overlaps = (start: string, end: string, from?: string, to?: string) =>
  (!to || start <= to) && (!from || end >= from)
```

Search is accent-insensitive (`Győr` matches `gyor`).
Date range filters check every occurrence of a recurring conference.

## Create conference flow

Goal: the user lands back on the previous page and the new card is already there. Then a notification.

The trap: `invalidateQueries` only refetches queries on screen. On `/conferences/new` the list is not on screen.
The list loader uses `staleTime: 'static'`, which returns cached data even when invalidated.
Result without the fix: old list first, new card pops in later.

The fix:
1. `mutationFn` does all writes: missing tags, the conference, the ConferenceTags rows.
2. Global `MutationCache.onSuccess` awaits invalidation with `refetchType: 'all'`.
3. The component's `onSuccess` shows the notification and goes back.
4. Order is guaranteed: cache callbacks run and are awaited before the mutation's own callbacks. `isPending` covers everything.

```ts
// src/features/conferences/api.ts
export async function createConference(input: ConferenceFormValues) {
  const tagIds = await ensureTagIds(input.tags) // creates missing tags first
  const { id } = await conferencesApi.add(toConferenceWrite(input))
  const results = await Promise.allSettled(
    tagIds.map((tagId) => conferenceTagsApi.add({ conference: id, tag: tagId })),
  )
  const tagsFailed = results.some((r) => r.status === 'rejected')
  if (tagsFailed) spLog.warn('ConferenceTags failed', { id, results })
  return { id, tagsFailed }
}
```

The conference is kept even if tag rows fail. That's the agreed behavior.

```ts
// src/features/conferences/queries.ts
import { mutationOptions } from '@tanstack/react-query'

export const conferenceMutations = {
  create: () =>
    mutationOptions({
      mutationKey: ['conferences', 'create'],
      mutationFn: createConference,
      meta: {
        invalidates: [conferenceKeys.all, conferenceTagKeys.all, tagKeys.all],
        errorTitle: 'Could not add conference',
      },
    }),
}
```

```tsx
// src/features/conferences/components/ConferenceCreatePage.tsx
const goBack = useGoBack()
const saved = useRef(false) // see "Unsaved changes" in router.md

const create = useMutation({
  ...conferenceMutations.create(),
  onSuccess: ({ tagsFailed }) => {
    notifications.show(
      tagsFailed
        ? { color: 'yellow', title: 'Conference saved', message: 'Tags could not be saved. Edit the conference to add them.' }
        : { color: 'green', message: 'Conference added' },
    )
    saved.current = true
    goBack()
  },
})

<form onSubmit={form.onSubmit((values) => create.mutate(values))}>
  …
  <Button type="submit" loading={create.isPending}>Save</Button>
</form>
```

Put `onSuccess` on `useMutation`, not on `mutate(…, { onSuccess })`. Per-call callbacks can be skipped after unmount.

Not chosen: writing the new item into the cache from the POST response. It means rebuilding lookup titles, tags, `Created`, `Author` by hand. One refetch is cheaper to maintain.

## Edit with ETag

Each item carries its `etag`. Updates send `IF-MATCH: <etag>`, never `*`.
A 412 means someone else changed the item.

- The global toast shows the message (SpError maps 412 to a readable text).
- The form shows a "Reload" button when `update.error?.status === 412`. Reload refetches and resets the form.
- Tags: diff old and new tag ids. Add new rows, delete removed rows. Same `allSettled` pattern.

## Delete

One call. ConferenceTags and Registrations rows go via cascade delete on their lookup (see [sharepoint-lib.md](./sharepoint-lib.md#lists)).
The detail page renders nothing when `useConferenceView(id)` is undefined. `onSuccess` then navigates to `/conferences`.

## Follow / like (optimistic)

Instant UI. Roll back on error. Disable the button while its toggle is pending.
Invalidate only when the last toggle settles, so one toggle doesn't revert another.

```ts
// src/features/follows/queries.ts
export const followMutations = {
  toggle: (target: FollowTarget) =>
    mutationOptions({
      mutationKey: ['follows', 'toggle'],
      mutationFn: async (existing: Follow | undefined) => {
        if (existing) await followsApi.remove(existing.id)
        else await followsApi.add(toFollowWrite(target))
      },
      onMutate: async (existing, { client }) => {
        const key = followQueries.mine().queryKey // typed key → typed cache data
        await client.cancelQueries({ queryKey: key })
        const previous = client.getQueryData(key)
        client.setQueryData(key, (old = []) =>
          existing ? old.filter((f) => f.id !== existing.id) : [...old, optimisticFollow(target)],
        )
        return { previous }
      },
      onError: (_e, _v, result, { client }) => client.setQueryData(followQueries.mine().queryKey, result?.previous),
      onSettled: (_d, _e, _v, _r, { client }) =>
        client.isMutating({ mutationKey: ['follows', 'toggle'] }) === 1
          ? client.invalidateQueries({ queryKey: followKeys.all })
          : undefined,
    }),
}
```

No `meta.invalidates` here. The optimistic flow handles it.

## Forms (Mantine)

- Render the form only when data exists (suspense guarantees it).
- Server data sets the initial values once. After that it's form state. Don't sync refetches into it.
- Validate with the Zod form schema from `model.ts` (Mantine form resolver; check which export supports Zod 4 in Mantine 9).
- Disable submit with `isPending`.

## Spotlight data

- Local state: query text + filters. Debounce the text (`useDebouncedValue`, about 200 ms).
- Results come from `useJoinedConferences()` and the same `derive.ts` functions as the list page.
- Sections: matches, recently added, upcoming, tags.
- Wrap the results in their own `<Suspense>`. The navbar must never suspend.
- No SharePoint Search API for conferences. It only shows new items after a crawl.

People search (lib is ready, UI is out of scope):

```ts
// src/features/users/queries.ts
export const peopleQueries = {
  search: (q: string) =>
    queryOptions({
      queryKey: ['people', 'search', q],
      queryFn: q.length >= 3 ? ({ signal }) => searchPeople(q, { signal }) : skipToken,
      staleTime: 5 * 60_000,
    }),
}
```

`skipToken` works with `useQuery`, not with `useSuspenseQuery`.

## Server-side mode later

What changes:
- `useConferenceList` internals → `useQuery({ ...conferenceQueries.list(filters), placeholderData: keepPreviousData })`.
- The list query builds OData `$filter` / `$orderby` / `$top` from the filters.
- Route gets `loaderDeps` with the filter params. Loader primes the filtered query.
- Paging becomes cursor-based ("Load more", `useInfiniteQuery`). No page numbers.
- Tags/registrations for the visible page: `ConferenceId eq 1 or …` in chunks (URL length limit).
- Filtered columns must be indexed (5,000 item threshold).

Components don't change. That's the point of the boundary.

## TanStack DB (consideration)

What it is: client-side collections with live queries (joins, filters, sorts) and optimistic transactions.
Why it fits: our joins (conference ↔ tags ↔ registrations ↔ follows) and in-memory filtering are its core job.
Its `on-demand` sync mode pushes `where` / `orderBy` / `limit` to your loader. That could map to OData later.

Status: `@tanstack/react-db` 0.5 (pre-1.0). `@tanstack/query-db-collection` 1.4 plugs into the existing QueryClient.
The docs say refetch-after-write behavior changes at 1.0.

Adoption path, if we do it:
1. One collection per list, built on the same `api.ts` functions and the same QueryClient.
2. Replace the inside of `useJoinedConferences` / `useConferenceList` with live queries.
3. Components stay the same.

Sketch only. Check the DB docs for the current API:

```ts
import { createCollection } from '@tanstack/react-db'
import { queryCollectionOptions } from '@tanstack/query-db-collection'

export const conferencesCollection = createCollection(
  queryCollectionOptions({
    queryKey: conferenceKeys.items(),
    queryFn: () => getConferences(),
    queryClient,
    getKey: (c) => c.id,
  }),
)
```

Consider it when `derive.ts` gets heavy, or when server-side mode starts.

## Gotchas

- `invalidateQueries` refetches only active queries unless `refetchType: 'all'`.
- `staleTime: 'static'` ignores invalidation. Fine in loaders, because writes await a real refetch.
- Return or await invalidation. Otherwise `isPending` ends too early.
- For `getQueryData` / `setQueryData`, use `xQueries.y().queryKey`. It's typed. A plain key array gives `unknown`.
- Don't parse or throw in `select`. Parse in the query function.
- On a background refetch error, keep showing the old data.
- Never create a `QueryClient` inside a component render.
