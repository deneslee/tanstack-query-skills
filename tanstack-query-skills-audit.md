# TanStack Query skills audit

Source: `SKILLS/tanstack-query` = `@tanstack/query-intent` 5.101.0, 29 skills.
Checked against `@tanstack/react-query` 5.104.5 (2026-10-20).

Where a skill and our docs disagree, our docs win (for example `refetchType: 'all'`, no "derived query").

## Deprecated APIs

All marked "will be removed in the next major version".

| In the skills | Use instead |
| --- | --- |
| `queryClient.ensureQueryData(o)` | `queryClient.query({ ...o, staleTime: 'static' })` |
| `queryClient.fetchQuery(o)` | `queryClient.query(o)` |
| `queryClient.prefetchQuery(o)` | `queryClient.query(o).catch(noop)` |
| `queryClient.fetchInfiniteQuery(o)` | `queryClient.infiniteQuery(o)` |
| `queryClient.prefetchInfiniteQuery(o)` | `queryClient.infiniteQuery(o).catch(noop)` |
| `queryClient.ensureInfiniteQueryData(o)` | `queryClient.infiniteQuery({ ...o, staleTime: 'static' })` |

Not deprecated: `usePrefetchQuery`, `usePrefetchInfiniteQuery`, `queryOptions`, `mutationOptions`.
The skills never mention `queryClient.query()`. Mentally replace as above when reading them.

Lines with deprecated calls, per skill:

| Skill | Lines |
| --- | --- |
| lifecycle/prefetch-and-remove-request-waterfalls | 13 |
| compositions/compose-query-with-tanstack-router-and-start | 7 |
| lifecycle/ssr-hydration-and-streaming | 5 |
| core/fetch-and-observe-queries | 3 |
| lifecycle/test-query-code | 3 |
| core/design-query-keys-and-options | 1 |
| core/understand-query-internals-and-observers | 1 |
| lifecycle/setup-query-client-and-providers | 1 |
| lifecycle/migrate-major-versions-and-codemods | 1 |

Other old APIs (`cacheTime`, `keepPreviousData: true`, `useErrorBoundary`) only appear in "Wrong" examples. Fine.

## Verdict per skill

**Use** = relevant as is. **Partly** = relevant, but read with the notes. **Later** = not needed yet. **Skip** = not relevant for a React SPA without SSR.

| Skill | Verdict | Notes |
| --- | --- | --- |
| compositions/automatic-invalidation-after-mutations | Use | Basis of `meta.invalidates`. We add `refetchType: 'all'`. |
| compositions/broadcast-realtime-and-multi-tab-synchronization | Skip | No realtime from SharePoint. Broadcast is experimental. |
| compositions/compose-query-with-tanstack-router-and-start | Partly | Loader + `useSuspenseQuery` is right. Uses `ensureQueryData`. Skip SSR/Start parts. Prefer the Router repo's `router-query` skill. |
| compositions/enforce-query-best-practices-with-eslint | Use | Enable `flat/recommended-strict`. |
| compositions/persist-offline-and-restore-caches | Later | Only if we want instant startup from storage. |
| compositions/query-data-and-forms | Use | Initial values once, no syncing into the form. |
| core/build-query-abstractions | Use | Factories first, hooks second. |
| core/cancel-queries-and-consume-abort-signals | Use | Pass `signal` to `fetch`. Ignore the axios parts. |
| core/concurrent-optimistic-updates | Use | Follow/like toggles (`isMutating` guard). |
| core/coordinate-query-execution | Use | `useQueries`, `skipToken`, background fetch indicators. |
| core/design-query-keys-and-options | Use | One deprecated call in a preload example. |
| core/fetch-and-observe-queries | Partly | Deprecated calls. Ignore `createQuery` / `injectQuery` (Solid/Angular). |
| core/implement-optimistic-updates-and-cache-writes | Use | Skip offline/persister parts. |
| core/paginate-and-build-infinite-queries | Later | Server-side mode (cursor paging, "Load more"). |
| core/seed-placeholder-select-and-transform-data | Use | `select`; `keepPreviousData` for server mode. Skip hydration bits. |
| core/selectors-and-derived-state | Use | Derive during render, don't sync with effects. |
| core/tune-defaults-freshness-retries-and-refetching | Use | `staleTime` vs `gcTime`, retry rules. |
| core/understand-query-internals-and-observers | Use | Reference. Explains active vs inactive queries (why `refetchType: 'all'`). |
| core/write-mutations-and-invalidate-related-queries | Use | Hook-level callbacks, awaited invalidation. |
| framework/debug-with-devtools | Partly | React parts only. Devtools are already set up. |
| framework/handle-status-and-errors | Use | Toast only background errors. |
| framework/shape-data-and-render-efficiently | Partly | React parts only. Skip Vue sections. |
| framework/use-framework-adapter-reactivity | Skip | Vue/Solid/Svelte/Angular/Lit only. |
| framework/use-suspense-and-error-boundaries | Use | Skip streamed hydration parts. |
| lifecycle/migrate-major-versions-and-codemods | Skip | We start on v5. |
| lifecycle/prefetch-and-remove-request-waterfalls | Partly | Ideas are right. Most examples use deprecated calls. |
| lifecycle/setup-query-client-and-providers | Partly | React part only. Skip SSR and other adapters. |
| lifecycle/ssr-hydration-and-streaming | Skip | No SSR. |
| lifecycle/test-query-code | Partly | React part only. Deprecated calls. Write fewer tests, see [testing.md](./testing.md). |

Summary: 16 Use, 7 Partly, 2 Later, 4 Skip.

## Router skills (not in this folder)

The TanStack Router repo ships its own skills. They are newer on the loader pattern (`queryClient.query()`).

| Path in TanStack/router | Use for |
| --- | --- |
| packages/router-core/skills/router-core/SKILL.md | entry point |
| …/router-core/search-params | Zod 4 search schemas, middlewares |
| …/router-core/data-loading | loaders, pending, errors, context |
| …/router-core/navigation | Link, navigate, preloading, blocking |
| …/router-core/path-params | `params.parse` / `stringify` |
| …/router-core/not-found-and-errors | `notFound()`, error components |
| …/router-core/type-safety | Register, inference |
| …/router-core/code-splitting | automatic code splitting |
| packages/react-router/skills/compositions/router-query | Router + Query (skip its SSR sections) |
| packages/router-plugin/skills/router-plugin | Vite plugin options |

Skip: `router-core/ssr`, `router-core/auth-and-guards` (Windows auth, no login flow), `lifecycle/migrate-from-react-router`.
