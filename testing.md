# Testing

Keep it small. Vitest 5. Node environment by default. jsdom only for the few flow tests.

## The rule

Write a test only if you can name the bug it prevents.
If you can't, don't write it.

## What to test: pure logic

Fast, stable, no mocks. Most tests live here.

| Test | Bug it prevents |
| --- | --- |
| `toDateOnly` / `fromDateOnly` around 2026-10-25 and 2027-03-28 | conference shown one day early or late |
| `lit.str("O'Brien")` | 400 error from SharePoint |
| `expandOccurrences`: monthly from Jan 31, yearly from Feb 29, count vs end date | wrong or missing calendar entries |
| `queryConferences`: date range overlap, tag filter, accent-insensitive search | conferences missing from results |
| search schema with junk URL values | crash when opening an old or edited link |
| field kinds: verbose vs JSON light arrays, empty lookup | parse failures on real data |

```ts
// src/lib/sharepoint/dates.test.ts
test('date-only survives the October clock change', () => {
  configureSharePoint({ webUrl: '/sites/x', timeZone: 'Europe/Budapest' })
  expect(toDateOnly('2026-10-24T22:00:00Z')).toBe('2026-10-25') // UTC+2 before the switch
  expect(toDateOnly('2026-10-25T23:00:00Z')).toBe('2026-10-26') // UTC+1 after the switch
  expect(fromDateOnly('2026-10-26')).toBe('2026-10-25T23:00:00.000Z')
})
```

## After codegen

Three checks. No SharePoint needed for the first two.

1. `tsc --noEmit`: generated descriptors compile.
2. Fixture check: every exported item parses with its descriptor.
3. Debug page "Checks" panel: the same check functions against the live site.

```ts
// src/test/fixtures.test.ts
import { checkItems } from '@/lib/sharepoint/checks'
import { lists } from '@/test/lists' // { Conferences: conferencesList, Tags: tagsList, … }

const fixtures = import.meta.glob<{ value: unknown[] }>('./fixtures/*.json', { eager: true, import: 'default' })

test.each(Object.entries(lists))('%s fixtures parse', (title, list) => {
  const raw = fixtures[`./fixtures/${title}.json`]
  expect(raw, `missing fixture for ${title}`).toBeDefined()
  expect(checkItems(list, raw.value).filter((r) => r.level === 'error')).toEqual([])
})
```

`lib/sharepoint/checks.ts` holds pure functions. Tests and the debug page call the same code.

| Check | Debug page | Fixture test |
| --- | --- | --- |
| items parse with the descriptor | yes | yes |
| fields in the descriptor exist with the same type | yes | no (needs live fields) |
| new required field in SharePoint we don't write | yes | no |
| item count vs 5,000; filtered columns indexed | yes | no |
| digest, current user, site timezone vs config | yes | no |

Fixtures contain real names and emails. Anonymizing the export is an open item.

## Flow tests with MSW

Only flows with real logic. Two or three tests in total.

1. Create conference → back to the previous page → new card is there → success notification.
2. ConferenceTags POST fails → conference is still listed → warning notification.
3. Optional: edit with a stale ETag → 412 → "Reload" button.

What MSW can and can't do:
- Can: response shapes, ETag, `odata.nextLink`, status codes, digest, a small in-memory store so create → refetch shows the item.
- Can't: OData filter behavior, thresholds, permissions, lookup limits, Windows login.
- So: replay real fixtures plus a small store. Never build an OData engine in MSW. The debug page checks the real behavior.

MSW v3 notes: ESM only, Node 22+, TypeScript 5.9+. `onUnhandledRequest` was renamed `onUnhandledFrame` (check the docs for the exact options).

One catch-all handler. SharePoint URLs contain `('…')`, so route them with our own parser, not MSW path patterns.

```ts
// src/test/msw/sharepoint.ts
import { http, HttpResponse } from 'msw'

export const spHandlers = (store: SpStore) => [
  http.post(/\/_api\/contextinfo$/, () =>
    HttpResponse.json({ FormDigestValue: 'test-digest', FormDigestTimeoutSeconds: 1800 }),
  ),
  http.all(/\/_api\//, ({ request }) => store.handle(request)),
]
```

The store (`createSpStore(fixtures)`):
- GET items: all items of the list, one page. `currentuser` returns a fixed user.
- POST add: new `Id`, etag `"1"`, returns the verbose shape.
- MERGE: compares `IF-MATCH` with the stored etag → 412 on mismatch.
- DELETE: removes the item.
- `store.failNext('ConferenceTags')`: the next write to that list returns 500.

Test setup:

```ts
// src/app/queryClient.ts exports a factory, so tests get the same caches and notifications
export const createQueryClient = (opts?: { retry?: false }) => new QueryClient({ /* … same config … */ })

// in a test
const queryClient = createQueryClient({ retry: false })
const router = createRouter({
  routeTree,
  history: createMemoryHistory({ initialEntries: ['/conferences', '/conferences/new'] }),
  context: { queryClient },
})
```

- New QueryClient and new store per test.
- `retry: false`, or failing tests wait for retries.
- Start the history with two entries, so "go back" has somewhere to go.
- Assert what the user sees (card title, notification text), not cache contents.

## Don't test

- Component snapshots.
- Mantine rendering or styling.
- TanStack Query caching or TanStack Router navigation themselves.
- Query key shapes.
- Mocks that repeat the implementation.
- Coverage numbers.

## Later, maybe

Playwright against the real site from a domain machine (Windows login works in Edge/Chromium). Release smoke test only.
