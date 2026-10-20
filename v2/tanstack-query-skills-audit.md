# TanStack skills audit

Query skills: shared copy in `SKILLS/tanstack-query`, package `query-skills` (from `@tanstack/query-intent` 5.101.0), 29 skills.
Checked against `@tanstack/react-query` 5.104.5 (2026-10-20). The 9 affected skills are updated in `query-skills.zip`.
Router, devtools and match-sorter skills come from npm via TanStack Intent.

Where a skill and our docs disagree, our docs win (for example `refetchType: 'all'`, no "derived query").

## Deprecated Query APIs

All marked "will be removed in the next major version".

| Old                                      | New                                                        |
| ---------------------------------------- | ---------------------------------------------------------- |
| `queryClient.ensureQueryData(o)`         | `queryClient.query({ ...o, staleTime: 'static' })`         |
| `queryClient.fetchQuery(o)`              | `queryClient.query(o)`                                     |
| `queryClient.prefetchQuery(o)`           | `queryClient.query(o).catch(noop)`                         |
| `queryClient.fetchInfiniteQuery(o)`      | `queryClient.infiniteQuery(o)`                             |
| `queryClient.prefetchInfiniteQuery(o)`   | `queryClient.infiniteQuery(o).catch(noop)`                 |
| `queryClient.ensureInfiniteQueryData(o)` | `queryClient.infiniteQuery({ ...o, staleTime: 'static' })` |

Not deprecated: `usePrefetchQuery`, `usePrefetchInfiniteQuery`, `queryOptions`, `mutationOptions`.
`query()` also accepts `select`, and does not retry unless `retry` is set.

## What changed in the updated copy

Every deprecated call in a code example is replaced. The only remaining old calls are in new "Wrong" examples that teach the change.
All changed code examples type-check against 5.104.5.

| Skill                                                     | Change                                                                                                                                                                                                                  |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| compositions/compose-query-with-tanstack-router-and-start | Description updated. 5 loader examples use `query({ …, staleTime: 'static' })`. Heading renamed. New mistake: "MEDIUM Deprecated ensureQueryData in loaders".                                                           |
| core/design-query-keys-and-options                        | `preloadUser` uses `query({ …, staleTime: 'static' })`.                                                                                                                                                                 |
| core/fetch-and-observe-queries                            | Description updated. Both imperative examples use `query()`, with one-line explanations. New mistake: "HIGH Using deprecated imperative fetch methods".                                                                 |
| core/understand-query-internals-and-observers             | Orchestration example uses `query({ …, staleTime: 'static' })`.                                                                                                                                                         |
| lifecycle/migrate-major-versions-and-codemods             | `prefetchTodo` uses `query(…).catch(noop)`.                                                                                                                                                                             |
| lifecycle/prefetch-and-remove-request-waterfalls          | Description updated. "Pick the right client method" → "One client method: query()". "Expecting prefetchQuery to return data" → "Using deprecated prefetch, fetch and ensure methods". All other examples use `query()`. |
| lifecycle/setup-query-client-and-providers                | Integration example uses `query({ …, staleTime: 'static' })`.                                                                                                                                                           |
| lifecycle/ssr-hydration-and-streaming                     | 4 server prefetches use `query(…).catch(noop)`, 1 `fetchQuery` → `query`.                                                                                                                                               |
| lifecycle/test-query-code                                 | 3 `fetchQuery` → `query`.                                                                                                                                                                                               |

Other old APIs (`cacheTime`, `keepPreviousData: true`, `useErrorBoundary`) only appear in "Wrong" examples. Fine.

## Verdict per Query skill

**Use** = relevant as is. **Partly** = relevant, read with the note. **Later** = excluded for now. **Skip** = not relevant for a React SPA without SSR (excluded).

| Skill                                                         | Verdict | Notes                                                               |
| ------------------------------------------------------------- | ------- | ------------------------------------------------------------------- |
| compositions/automatic-invalidation-after-mutations           | Use     | Basis of `meta.invalidates`. We add `refetchType: 'all'`.           |
| compositions/broadcast-realtime-and-multi-tab-synchronization | Skip    | No realtime from SharePoint. Broadcast is experimental.             |
| compositions/compose-query-with-tanstack-router-and-start     | Partly  | Loader patterns are right (updated). Skip the SSR/Start parts.      |
| compositions/enforce-query-best-practices-with-eslint         | Use     | Enable `flat/recommended-strict`.                                   |
| compositions/persist-offline-and-restore-caches               | Later   | Only if we want instant startup from storage.                       |
| compositions/query-data-and-forms                             | Use     | Initial values once, no syncing into the form.                      |
| core/build-query-abstractions                                 | Use     | Factories first, hooks second.                                      |
| core/cancel-queries-and-consume-abort-signals                 | Use     | Pass `signal` to `fetch`. Ignore the axios parts.                   |
| core/concurrent-optimistic-updates                            | Use     | Follow/like toggles (`isMutating` guard).                           |
| core/coordinate-query-execution                               | Use     | `useQueries`, `skipToken`, background fetch indicators.             |
| core/design-query-keys-and-options                            | Use     |                                                                     |
| core/fetch-and-observe-queries                                | Partly  | Ignore `createQuery` / `injectQuery` (Solid/Angular).               |
| core/implement-optimistic-updates-and-cache-writes            | Use     | Skip offline/persister parts.                                       |
| core/paginate-and-build-infinite-queries                      | Later   | Server-side mode (cursor paging, "Load more").                      |
| core/seed-placeholder-select-and-transform-data               | Use     | `select`; `keepPreviousData` for server mode.                       |
| core/selectors-and-derived-state                              | Use     | Derive during render, don't sync with effects.                      |
| core/tune-defaults-freshness-retries-and-refetching           | Use     | `staleTime` vs `gcTime`, retry rules.                               |
| core/understand-query-internals-and-observers                 | Use     | Explains active vs inactive queries (why `refetchType: 'all'`).     |
| core/write-mutations-and-invalidate-related-queries           | Use     | Hook-level callbacks, awaited invalidation.                         |
| framework/debug-with-devtools                                 | Partly  | React parts only. Devtools are already set up.                      |
| framework/handle-status-and-errors                            | Use     | Toast only background errors.                                       |
| framework/shape-data-and-render-efficiently                   | Partly  | React parts only. Skip Vue sections.                                |
| framework/use-framework-adapter-reactivity                    | Skip    | Vue/Solid/Svelte/Angular/Lit only.                                  |
| framework/use-suspense-and-error-boundaries                   | Use     | Skip streamed hydration parts.                                      |
| lifecycle/migrate-major-versions-and-codemods                 | Skip    | We start on v5.                                                     |
| lifecycle/prefetch-and-remove-request-waterfalls              | Use     | Updated.                                                            |
| lifecycle/setup-query-client-and-providers                    | Partly  | React part only. Skip SSR and other adapters.                       |
| lifecycle/ssr-hydration-and-streaming                         | Skip    | No SSR.                                                             |
| lifecycle/test-query-code                                     | Partly  | React part only. Write fewer tests, see [testing.md](./testing.md). |

Summary: 17 Use, 6 Partly, 2 Later, 4 Skip.

## Other Intent skills in the repo

| Skill id                                                            | Verdict | Notes                                                                                                    |
| ------------------------------------------------------------------- | ------- | -------------------------------------------------------------------------------------------------------- |
| `@tanstack/router-core#router-core` (entry) + 7 sub-skills          | Use     | search-params, data-loading, navigation, path-params, not-found-and-errors, type-safety, code-splitting. |
| `@tanstack/router-core#router-core/ssr`                             | Skip    | No SSR.                                                                                                  |
| `@tanstack/router-core#router-core/auth-and-guards`                 | Skip    | Windows auth, no login flow.                                                                             |
| `@tanstack/router-plugin#router-plugin`                             | Use     | Vite plugin options. Install of `@tanstack/router-plugin` brings it.                                     |
| `@tanstack/match-sorter-utils#fuzzy-ranking`                        | Use     | Spotlight / conference search. Has a diacritics option (check it for Hungarian names).                   |
| `@tanstack/devtools#devtools-app-setup`, `#devtools-production`     | Later   | Already set up. Re-enable when changing devtools or the production build.                                |
| `@tanstack/devtools-vite#devtools-vite-plugin`                      | Later   | Same.                                                                                                    |
| `@tanstack/devtools#devtools-marketplace`, `#devtools-plugin-panel` | Skip    | For devtools plugin authors.                                                                             |
| `@tanstack/devtools-event-client` (3 skills)                        | Skip    | For library authors.                                                                                     |

Not on npm yet: `@tanstack/react-router` 1.170.45 ships no skills. GitHub `main` already has them (`react-router`, `compositions/router-query`) and lists `skills` in `files`, so a later release will likely ship them.
Until then, the updated `compositions/compose-query-with-tanstack-router-and-start` covers loaders + Query. See [repo-setup.md](./repo-setup.md#8-router-skills).

The exclude list that matches these verdicts is in [repo-setup.md](./repo-setup.md#2-packagejson).
