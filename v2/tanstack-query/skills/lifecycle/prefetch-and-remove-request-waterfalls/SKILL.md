---
name: lifecycle/prefetch-and-remove-request-waterfalls
description: >
  Use this when flattening request waterfalls with queryClient.query and
  queryClient.infiniteQuery (they replace the deprecated prefetchQuery,
  prefetchInfiniteQuery, ensureQueryData and fetchQuery), usePrefetchQuery,
  usePrefetchInfiniteQuery, route loaders, TanStack Router loaders, Start routes,
  Suspense prefetching, and query-function prefetching.
type: lifecycle
library: TanStack Query
library_version: '5.101.0'
requires:
  - core/design-query-keys-and-options
  - core/fetch-and-observe-queries
  - core/tune-defaults-freshness-retries-and-refetching
sources:
  - TanStack/query:docs/framework/react/guides/prefetching.md
  - TanStack/query:docs/framework/react/guides/request-waterfalls.md
  - TanStack/query:docs/framework/react/reference/usePrefetchQuery.md
  - TanStack/query:docs/framework/react/reference/usePrefetchInfiniteQuery.md
  - TanStack/query:docs/reference/QueryClient.md
  - TanStack/query:examples/react/react-router/src/routes/root.tsx
  - TanStack/query:examples/react/react-router/src/routes/contact.tsx
---

## Setup

```ts
import { noop, QueryClient, queryOptions } from '@tanstack/react-query'

const queryClient = new QueryClient()

const todosOptions = queryOptions({
  queryKey: ['todos'],
  queryFn: async () => [{ id: 1, title: 'Ship' }],
  staleTime: 60_000,
})

export function preloadTodos() {
  return queryClient.query(todosOptions).catch(noop)
}
```

## Core Patterns

### Use loaders for route-critical data

```ts
import { QueryClient, queryOptions } from '@tanstack/react-query'

const queryClient = new QueryClient()
const contactOptions = (contactId: string) =>
  queryOptions({
    queryKey: ['contact', contactId],
    queryFn: async () => ({ id: contactId, name: 'Ada' }),
  })

export function contactLoader(contactId: string) {
  return queryClient.query({ ...contactOptions(contactId), staleTime: 'static' })
}
```

### Prefetch before Suspense can suspend

```tsx
import { Suspense } from 'react'
import { usePrefetchQuery, useSuspenseQuery } from '@tanstack/react-query'

const commentsOptions = {
  queryKey: ['comments'],
  queryFn: async () => [{ id: 1 }],
}

function Comments() {
  const { data } = useSuspenseQuery(commentsOptions)
  return <pre>{JSON.stringify(data)}</pre>
}

export function CommentsSection() {
  usePrefetchQuery(commentsOptions)
  return (
    <Suspense fallback={<p>Loading</p>}>
      <Comments />
    </Suspense>
  )
}
```

### One client method: query()

```ts
import { noop, QueryClient, queryOptions } from '@tanstack/react-query'

const queryClient = new QueryClient()
const todoOptions = (id: string) =>
  queryOptions({ queryKey: ['todo', id], queryFn: async () => ({ id }) })

// Needs the data or the error
export function getTodo(id: string) {
  return queryClient.query(todoOptions(id))
}

// Only warms the cache
export function warmTodo(id: string) {
  void queryClient.query(todoOptions(id)).catch(noop)
}

// Uses any cached data, fetches only when the cache is empty (loaders)
export function ensureTodo(id: string) {
  return queryClient.query({ ...todoOptions(id), staleTime: 'static' })
}
```

`query()` replaces `fetchQuery`, `prefetchQuery` and `ensureQueryData`. `infiniteQuery()` does the same for infinite queries.

## Common Mistakes

### HIGH Using deprecated prefetch, fetch and ensure methods

Wrong:

```ts
import { QueryClient } from '@tanstack/react-query'

const queryClient = new QueryClient()
const options = { queryKey: ['todo', 1], queryFn: async () => ({ id: 1 }) }

await queryClient.prefetchQuery(options)
const data = await queryClient.fetchQuery(options)
const cached = await queryClient.ensureQueryData(options)
```

Correct:

```ts
import { noop, QueryClient } from '@tanstack/react-query'

const queryClient = new QueryClient()
const options = { queryKey: ['todo', 1], queryFn: async () => ({ id: 1 }) }

await queryClient.query(options).catch(noop)
const data = await queryClient.query(options)
const cached = await queryClient.query({ ...options, staleTime: 'static' })
```

These methods (and their infinite variants) are deprecated and will be removed in the next major version. Add `.catch(noop)` only where the caller should not see errors.

Source: TanStack/query:docs/reference/QueryClient.md

### HIGH Prefetch staleTime only set on prefetch

Wrong:

```ts
import { noop, QueryClient, useQuery } from '@tanstack/react-query'

const queryClient = new QueryClient()
await queryClient
  .query({
    queryKey: ['todos'],
    queryFn: async () => [{ id: 1 }],
    staleTime: 60_000,
  })
  .catch(noop)
export function useTodos() {
  return useQuery({ queryKey: ['todos'], queryFn: async () => [{ id: 1 }] })
}
```

Correct:

```ts
import { noop, QueryClient, queryOptions, useQuery } from '@tanstack/react-query'

const queryClient = new QueryClient()
const options = queryOptions({
  queryKey: ['todos'],
  queryFn: async () => [{ id: 1 }],
  staleTime: 60_000,
})
await queryClient.query(options).catch(noop)
export function useTodos() {
  return useQuery(options)
}
```

Options passed to a one-off `query()` call do not configure the later observer.

Source: TanStack/query:docs/framework/react/guides/prefetching.md

### HIGH Suspense prefetch after suspension

Wrong:

```tsx
import * as React from 'react'
import { noop, useQueryClient, useSuspenseQuery } from '@tanstack/react-query'

export function Article() {
  const queryClient = useQueryClient()
  const article = useSuspenseQuery({
    queryKey: ['article'],
    queryFn: async () => ({ id: 1 }),
  })
  React.useEffect(() => {
    void queryClient
      .query({
        queryKey: ['comments'],
        queryFn: async () => [],
      })
      .catch(noop)
  }, [queryClient])
  return <pre>{JSON.stringify(article.data)}</pre>
}
```

Correct:

```tsx
import { Suspense } from 'react'
import { usePrefetchQuery, useSuspenseQuery } from '@tanstack/react-query'

function Article() {
  const article = useSuspenseQuery({
    queryKey: ['article'],
    queryFn: async () => ({ id: 1 }),
  })
  return <pre>{JSON.stringify(article.data)}</pre>
}

export function ArticleRoute() {
  usePrefetchQuery({ queryKey: ['comments'], queryFn: async () => [] })
  return (
    <Suspense fallback={<p>Loading</p>}>
      <Article />
    </Suspense>
  )
}
```

Effects do not run until after a suspenseful query resolves, so they cannot flatten that waterfall.

Source: TanStack/query:docs/framework/react/guides/prefetching.md

See also: `compositions/compose-query-with-tanstack-router-and-start` for Router and Start loader prefetching.
