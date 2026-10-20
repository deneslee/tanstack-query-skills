# Conference app – agent guide

Conference discovery app. React SPA. Hosted as a static build on SharePoint 2016.
Data lives in SharePoint lists. We talk to SharePoint REST directly (no PnPjs).

Read this file first. Then open the topic file you need.

| File | Read when you work on |
| --- | --- |
| [router.md](./router.md) | routes, search params, loaders, navigation |
| [query.md](./query.md) | queries, mutations, cache, notifications, spotlight data |
| [sharepoint-lib.md](./sharepoint-lib.md) | `lib/sharepoint`, field descriptors, codegen, debug page, lists |
| [testing.md](./testing.md) | tests and MSW |
| [tanstack-query-skills-audit.md](./tanstack-query-skills-audit.md) | which TanStack Query skills to trust |

## Stack (latest as of 2026-10-20)

| Package | Version |
| --- | --- |
| @tanstack/react-query | 5.104 |
| @tanstack/react-router + router-plugin | 1.170 / 1.168 |
| zod | 4.6 |
| @mantine/* (core, dates, form, notifications, spotlight) | 9.7 |
| dayjs (+ utc, timezone plugins) | 1.11 |
| msw | 3.0 |
| vitest | 5.0 |
| TanStack DB (only a consideration) | @tanstack/react-db 0.5 |

Not used: axios, PnPjs, query-key-factory, `@tanstack/zod-adapter`.

## App in one paragraph

Home: recently added, upcoming, by category, tags.
Conferences page: search, filter dropdowns, sort, tag filter, date range. Three views: grid, table (both paginated), timeline.
Calendar page. Spotlight search in the navbar (Ctrl+K).
Any user can add, edit, delete conferences (MVP).
Users can follow categories, tags, organizations and like conferences.
Users request to attend a specific occurrence. Approval workflow is out of scope.

## Decisions

| Topic | Decision |
| --- | --- |
| History | Hash history. SharePoint can't rewrite URLs to `index.html`. |
| Hosting | `.html` file in a document library. Works today. |
| Data loading | Browser-side for now: load whole lists, join and filter in memory. Server-side later. |
| List boundary | `useConferenceList(filters)` returns `ListResult` = `{ items, total?, nextCursor? }`. Later server mode swaps its internals only. |
| Loaders | `queryClient.query({ ...opts, staleTime: 'static' })`. Components read with `useSuspenseQuery`. |
| After writes | Global `MutationCache` invalidates `meta.invalidates` with `refetchType: 'all'` and awaits it. |
| After save | Go back to the previous page (`useCanGoBack`), fallback `/conferences`. Success notification. |
| Tags | `Tags` list + `ConferenceTags` junction list. If tag rows fail: keep the conference, warn. |
| Category / Organization | Lookup lists. |
| Country | Choice column (MVP). |
| Presenters | SharePoint users (Person, multi). |
| Dates | Date-only. Domain type is a `'YYYY-MM-DD'` string. Site timezone is a config constant. |
| Recurrence | Rule on the conference item (frequency, interval, count and/or end date). Occurrences computed in the browser. |
| Registrations | Per occurrence: conference id + occurrence date. |
| Spotlight | Navbar overlay, local state. "See all" opens `/conferences` with the filters in the URL. |
| Validation | Zod 4 everywhere. SharePoint responses are parsed in `api.ts`/lib, never in components. |
| Codegen | Debug page exports list schema JSON. A Node script generates field descriptors (`*.gen.ts`). |
| TanStack DB | Consideration only. See [query.md](./query.md#tanstack-db-consideration). |

## Folder structure

```
src/
  app/              providers, queryClient.ts, router.ts, theme, styles
  components/       shared UI: FilterBar, ViewChanger, ItemGrid, AppSpotlight…
  lib/sharepoint/   portable SharePoint REST layer. No React, no TanStack, no Mantine.
  lib/dayjs.ts      configured dayjs (utc + timezone)
  lib/types.ts      shared types, e.g. ListResult<T>
  features/         one SharePoint list = one feature
    conferences/
      components/
      model.gen.ts  generated field descriptor – never edit
      model.ts      domain types, search schema, form schema
      api.ts        SharePoint calls → parsed domain objects
      queries.ts    queryOptions / mutationOptions factories, keys, hooks
      derive.ts     pure functions: join, filter, sort, paginate, occurrences
    tags/  conferenceTags/  registrations/  follows/  categories/  organizations/
    users/          current user, people search (no SharePoint list of its own)
  routes/           thin route files (file-based routing)
  test/             MSW store + handlers, fixtures/, setup.ts
scripts/
  sp-codegen.ts     schema JSON → model.gen.ts
```

## Hard rules for agents

1. `lib/sharepoint` never imports React, TanStack or Mantine.
2. Every server read goes through a `queryOptions` factory in `features/*/queries.ts`. No inline `useQuery({ queryKey… })` in components.
3. Loaders only prime the cache. Components read with `useSuspenseQuery` / `useQuery`. Never `useLoaderData` for server data.
4. Don't use deprecated Query APIs: `ensureQueryData`, `fetchQuery`, `prefetchQuery` (and the infinite versions). Use `queryClient.query()` / `infiniteQuery()`.
5. Search params: pass the Zod 4 schema straight to `validateSearch`. Use `.default().catch()`. No zod adapter.
6. Never annotate or cast inferred router types.
7. Every write is a `mutationOptions` factory with `meta.invalidates`.
8. Date-only values are `'YYYY-MM-DD'` strings in the domain. Convert only in `lib/sharepoint/dates.ts`.
9. Never edit `*.gen.ts` or `routeTree.gen.ts`. Regenerate them.
10. Write a test only if you can name the bug it prevents. See [testing.md](./testing.md).
11. Don't add dependencies (axios, PnPjs, query-key-factory, date libs) without asking.
12. Use `<Link>` / `navigate`, never `<a href="#/…">`.
