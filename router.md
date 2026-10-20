# TanStack Router

Router 1.170. File-based routing. Vite plugin with automatic code splitting.
Query owns server data. The router only coordinates loading.

## Setup

Hash history, because SharePoint can't rewrite every URL to `index.html`.
URLs look like `…/index.html#/conferences?view=table&page=2`.

```ts
// vite.config.ts
import { tanstackRouter } from '@tanstack/router-plugin/vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  base: './', // assets load relative to index.html in the SP library
  plugins: [tanstackRouter({ target: 'react', autoCodeSplitting: true }), react()],
})
```

```ts
// src/app/router.ts
import { createHashHistory, createRouter } from '@tanstack/react-router'
import { routeTree } from '../routeTree.gen'
import { queryClient } from './queryClient'

export const router = createRouter({
  routeTree,
  history: createHashHistory(),
  context: { queryClient },
  defaultPreload: 'intent',
  defaultPreloadStaleTime: 0, // let Query decide freshness
  scrollRestoration: true,
  defaultPendingComponent: PagePending,
  defaultErrorComponent: RouteError,
  defaultNotFoundComponent: NotFound,
})

declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}
```

Provider order in `main.tsx`: `MantineProvider` → `Notifications` → `QueryClientProvider` → `RouterProvider`.

## Root route and context

`createRootRouteWithContext` is a factory. Note the double call.
The current user is loaded once and cached forever.

```tsx
// src/routes/__root.tsx
import type { QueryClient } from '@tanstack/react-query'
import { createRootRouteWithContext } from '@tanstack/react-router'
import { currentUserQuery } from '@/features/users/queries'

interface RouterContext {
  queryClient: QueryClient
}

export const Route = createRootRouteWithContext<RouterContext>()({
  beforeLoad: async ({ context }) => ({
    currentUser: await context.queryClient.query({ ...currentUserQuery(), staleTime: 'static' }),
  }),
  component: AppLayout, // AppShell, navbar, spotlight, <Outlet />
})
```

## Route tree

```
routes/
  __root.tsx            layout + current user
  index.tsx             home
  conferences/
    index.tsx           list (search params, grid/table/timeline)
    new.tsx             create form
    $id/
      route.tsx         parses $id, throws notFound(), renders <Outlet />
      index.tsx         detail
      edit.tsx          edit form
  calendar.tsx          calendar (?month + shared filters)
  debug/
    index.tsx           lists overview (gated)
    $list.tsx           fields, items, add item, export, checks
```

Rename or move route files. Never edit the path string in `createFileRoute` by hand.

## Route files stay thin

A route file holds: `validateSearch`, `search` middlewares, `params`, `beforeLoad`, `loader`, `component`.
Keep that order. Types flow from earlier options into later ones.
The component lives in `features/`.

```tsx
// src/routes/conferences/index.tsx
import { createFileRoute, stripSearchParams } from '@tanstack/react-router'
import { conferenceSearchDefaults, conferenceSearchSchema } from '@/features/conferences/model'
import { primeConferenceData } from '@/features/conferences/queries'
import { ConferencesPage } from '@/features/conferences/components/ConferencesPage'

export const Route = createFileRoute('/conferences/')({
  validateSearch: conferenceSearchSchema,
  search: { middlewares: [stripSearchParams(conferenceSearchDefaults)] },
  // No loaderDeps: filters run in memory. The loader only primes the base lists.
  loader: async ({ context }) => {
    await primeConferenceData(context.queryClient)
  },
  component: ConferencesPage,
})
```

Loaders prime the cache with `queryClient.query({ ...opts, staleTime: 'static' })`.
Components read with `useSuspenseQuery`. Never use `useLoaderData` for server data.

## Search params (Zod 4)

Pass the schema straight to `validateSearch`. Zod 4 implements Standard Schema.
Use `.default()` for missing values and `.catch()` for junk values. No `@tanstack/zod-adapter`.

```ts
// src/features/conferences/model.ts
import { z } from 'zod'

export const conferenceSearchDefaults = {
  q: '',
  tags: [] as number[],
  sort: 'start-asc' as const,
  view: 'grid' as const,
  page: 1,
  pageSize: 24,
}
const d = conferenceSearchDefaults
const isoDate = z.iso.date() // 'YYYY-MM-DD'

export const conferenceSearchSchema = z.object({
  q: z.string().default(d.q).catch(d.q),
  tags: z.array(z.number()).default(d.tags).catch(d.tags),
  category: z.number().optional().catch(undefined),
  organization: z.number().optional().catch(undefined),
  country: z.string().optional().catch(undefined),
  from: isoDate.optional().catch(undefined),
  to: isoDate.optional().catch(undefined),
  sort: z.enum(['start-asc', 'start-desc', 'created-desc', 'title']).default(d.sort).catch(d.sort),
  view: z.enum(['grid', 'table', 'timeline']).default(d.view).catch(d.view),
  page: z.number().int().min(1).default(d.page).catch(d.page),
  pageSize: z.number().int().min(6).max(96).default(d.pageSize).catch(d.pageSize),
})

export type ConferenceSearch = z.infer<typeof conferenceSearchSchema>
```

Rules:
- `view` is a search param, not a route.
- Dates in the URL are `'YYYY-MM-DD'` strings. Never `Date` objects.
- Reset `page` to 1 whenever a filter changes.
- Use `replace: true` for typing and filter tweaks. Back should not replay every keystroke.
- Debounce the search box (about 300 ms) before writing to the URL.
- `/calendar` reuses the filter part of the schema (`.pick()` / `.extend({ month })`).

Feature components use `getRouteApi` (no import cycle with the route file).

```tsx
// src/features/conferences/components/TagFilter.tsx
import { getRouteApi } from '@tanstack/react-router'

const routeApi = getRouteApi('/conferences/')

export function TagFilter() {
  const { tags } = routeApi.useSearch()
  const navigate = routeApi.useNavigate()
  const setTags = (next: number[]) =>
    navigate({ to: '.', search: (prev) => ({ ...prev, tags: next, page: 1 }), replace: true })
  // render a Mantine MultiSelect with value={tags} onChange={setTags}
}
```

Use the function form `search: (prev) => …` to keep other params.
The object form replaces all params. Use it only on purpose (spotlight "See all", "Reset filters").

## Path params and not found

Parse `$id` once in the `$id` layout route. Children get a typed `number`.

```tsx
// src/routes/conferences/$id/route.tsx
import { createFileRoute, notFound } from '@tanstack/react-router'
import { z } from 'zod'
import { primeConferenceData } from '@/features/conferences/queries'

export const Route = createFileRoute('/conferences/$id')({
  params: {
    parse: ({ id }) => ({ id: z.coerce.number().int().positive().parse(id) }),
    stringify: ({ id }) => ({ id: String(id) }),
  },
  loader: async ({ context, params }) => {
    const [conferences] = await primeConferenceData(context.queryClient)
    if (!conferences.some((c) => c.id === params.id)) throw notFound()
  },
})
```

## Pending and errors

Set defaults on the router. Override per route only when needed.
Pending UI shows after 1 s by default (`pendingMs`). Keep the default.

The error component resets Query's error boundary and re-runs loaders.
`reset()` alone does not re-run the loader.

```tsx
// src/components/RouteError.tsx
import { useQueryErrorResetBoundary } from '@tanstack/react-query'
import { type ErrorComponentProps, useRouter } from '@tanstack/react-router'

export function RouteError({ error }: ErrorComponentProps) {
  const router = useRouter()
  const { reset } = useQueryErrorResetBoundary()
  useEffect(() => reset(), [reset])
  return (
    <Alert color="red" title="Something went wrong">
      {error.message}
      <Button onClick={() => router.invalidate()}>Retry</Button>
    </Alert>
  )
}
```

## After save: go back

Go back to wherever the user came from. If there is no history (direct link to `/conferences/new`), go to `/conferences`.

```ts
// src/components/useGoBack.ts
import { useCanGoBack, useRouter } from '@tanstack/react-router'

export function useGoBack() {
  const router = useRouter()
  const canGoBack = useCanGoBack()
  return () => (canGoBack ? router.history.back() : router.navigate({ to: '/conferences' }))
}
```

The data is already fresh when we go back. See the create flow in [query.md](./query.md#create-conference-flow).

## Unsaved changes

Use `useBlocker` on new/edit forms. Prefer `withResolver: true` with a Mantine modal over `confirm()`.

Trap: after a successful save the form is still dirty, so the blocker blocks your own "go back".
Set a ref before navigating:

```ts
const saved = useRef(false)
useBlocker({ shouldBlockFn: () => form.isDirty() && !saved.current, withResolver: true })
// on success: saved.current = true; goBack()
```

## Spotlight → conferences page

Spotlight state is local (not in the URL). "See all" navigates with the object form:

```ts
navigate({ to: '/conferences', search: { q, tags, country, from, to } })
spotlight.close()
```

## Debug route

Lazy by default (code splitting). Gate it in `beforeLoad`: dev mode, a localStorage flag, or a permission check. Throw `notFound()` otherwise.

## Gotchas

- `defaultPreloadStaleTime: 0` is required, or the router's 30 s preload cache hides Query's freshness.
- `loaderDeps` must return only what the loader uses. In browser mode there are none.
- Don't call hooks in `beforeLoad` or `loader`. Use router context.
- Don't annotate `useSearch()` / `useParams()` results. Types are inferred.
- Index route id has a trailing slash: `getRouteApi('/conferences/')`. Links use `to="/conferences"`.
- Server-side mode later: add `loaderDeps` with the filter params and prime the filtered query instead.
