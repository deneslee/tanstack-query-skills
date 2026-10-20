@AGENTS.md

# Rules

1. `lib/sharepoint` never imports React, TanStack or Mantine.
2. Every server read goes through a `queryOptions` factory in `features/*/queries.ts`. No inline `useQuery({ queryKey… })` in components.
3. Loaders only prime the cache. Components read with `useSuspenseQuery` / `useQuery`. Never `useLoaderData` for server data.
4. Don't use deprecated Query APIs: `ensureQueryData`, `fetchQuery`, `prefetchQuery` (and the infinite versions). Use `queryClient.query()` / `infiniteQuery()`.
5. Search params: pass the Zod 4 schema straight to `validateSearch`. Use `.default().catch()`. No zod adapter.
6. Never annotate or cast inferred router types.
7. Every write is a `mutationOptions` factory with `meta.invalidates`.
8. Date-only values are `'YYYY-MM-DD'` strings in the domain. Convert only in `lib/sharepoint/dates.ts`.
9. Never edit `*.gen.ts` or `routeTree.gen.ts`. Regenerate them.
10. Write a test only if you can name the bug it prevents. See `docs/agents/testing.md`.
11. Don't add dependencies (axios, PnPjs, query-key-factory, date libs) without asking.
12. Use `<Link>` / `navigate`, never `<a href="#/…">`.
