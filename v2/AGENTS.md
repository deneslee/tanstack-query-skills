# AGENTS.md

Conference discovery app. React SPA hosted on SharePoint 2016, data in SharePoint lists.
Rules are in `CLAUDE.md`. This file only routes you to the right doc or skill.

## Project docs

Read when the task touches the topic. Not upfront.
The big files (`query.md`, `sharepoint-lib.md`): search for the heading you need and read that section.

| Working on                                                      | Read                            |
| --------------------------------------------------------------- | ------------------------------- |
| stack, decisions, folder layout                                 | `docs/agents/README.md`         |
| routes, search params, loaders, navigation                      | `docs/agents/router.md`         |
| queries, mutations, notifications, spotlight data               | `docs/agents/query.md`          |
| `lib/sharepoint`, field descriptors, codegen, debug page, lists | `docs/agents/sharepoint-lib.md` |
| tests, MSW                                                      | `docs/agents/testing.md`        |

## Library skills (TanStack Intent)

Before editing code in one of these areas, load the matching skill:

```sh
pnpm run -s intent -- load "<skill id>"
```

- Use `pnpm run intent`, not `npx intent` or `pnpm exec intent`. `@tanstack/devtools` ships a broken `intent` command.
- Load one skill. Add a second only if the task really spans two areas.
- Project docs win over skills when they disagree.

| Task                                     | Skill id                                                                 |
| ---------------------------------------- | ------------------------------------------------------------------------ |
| route tree, createRouter, Register       | `@tanstack/router-core#router-core`                                      |
| search params                            | `@tanstack/router-core#router-core/search-params`                        |
| loaders, pending, errors, router context | `@tanstack/router-core#router-core/data-loading`                         |
| Link, navigate, preloading, useBlocker   | `@tanstack/router-core#router-core/navigation`                           |
| path params                              | `@tanstack/router-core#router-core/path-params`                          |
| notFound, error components               | `@tanstack/router-core#router-core/not-found-and-errors`                 |
| router type errors                       | `@tanstack/router-core#router-core/type-safety`                          |
| Vite plugin, code splitting              | `@tanstack/router-plugin#router-plugin`                                  |
| route loaders + Query                    | `query-skills#compositions/compose-query-with-tanstack-router-and-start` |
| keys, queryOptions, mutationOptions      | `query-skills#core/design-query-keys-and-options`                        |
| mutations, invalidation                  | `query-skills#core/write-mutations-and-invalidate-related-queries`       |
| global invalidation (`meta.invalidates`) | `query-skills#compositions/automatic-invalidation-after-mutations`       |
| optimistic updates                       | `query-skills#core/implement-optimistic-updates-and-cache-writes`        |
| select, derived data                     | `query-skills#core/selectors-and-derived-state`                          |
| loading/error states, toasts             | `query-skills#framework/handle-status-and-errors`                        |
| suspense, error boundaries               | `query-skills#framework/use-suspense-and-error-boundaries`               |
| staleTime, retries, refetching           | `query-skills#core/tune-defaults-freshness-retries-and-refetching`       |
| forms with server data                   | `query-skills#compositions/query-data-and-forms`                         |
| tests with Query                         | `query-skills#lifecycle/test-query-code`                                 |
| fuzzy search (spotlight)                 | `@tanstack/match-sorter-utils#fuzzy-ranking`                             |

No row fits? `pnpm run -s intent -- list` shows all allowed skills. Its output is large; use it rarely.
