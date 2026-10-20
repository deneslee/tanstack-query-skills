# lib/sharepoint

A portable SharePoint 2016 REST layer. Copy-paste it into any project.
Plain `fetch`. Zod for parsing. dayjs for dates. Nothing else.

## Rules

- No React, TanStack or Mantine imports. Enforce with ESLint `no-restricted-imports`.
- No retries here. Query retries. Two retry layers multiply.
- No notifications here. The app wires them through the Query caches ([query.md](./query.md#queryclient)).
- Same-origin only. Windows auth cookies are sent automatically.
- Only `configureSharePoint()` knows about the app.

## Files

```
lib/sharepoint/
  config.ts    configureSharePoint({ webUrl, timeZone, logger, strictParse })
  client.ts    spFetch, request digest, SpError
  log.ts       SpLogger, consoleLogger, spLog
  odata.ts     query string builder, literal escaping, listPath()
  fields.ts    field kinds (f.text, f.date, f.lookup…), defineList, parse/payload helpers
  list.ts      createListApi(list): getAll, add, update, remove, addAttachment
  people.ts    currentUser, searchPeople, ensureUser
  dates.ts     toDateOnly, fromDateOnly, todayDateOnly
  schema.ts    getLists, getFields, getContextInfo (debug page + codegen export)
  checks.ts    pure drift checks (tests + debug page)
  index.ts
```

## Config

```ts
// lib/sharepoint/config.ts
export type SpConfig = {
  webUrl: string // server-relative, e.g. '/sites/conferences'
  timeZone: string // IANA, e.g. 'Europe/Budapest'. Must match the site's regional settings.
  logger?: SpLogger
  strictParse?: boolean // dev/tests: throw on the first bad item
}

let config: SpConfig | undefined
export const configureSharePoint = (c: SpConfig) => { config = c }
export function spConfig(): SpConfig {
  if (!config) throw new Error('configureSharePoint() was not called')
  return config
}
```

```ts
// src/main.tsx (before render)
configureSharePoint({
  webUrl: import.meta.env.VITE_SP_WEB_URL,
  timeZone: 'Europe/Budapest',
  logger: consoleLogger,
  strictParse: import.meta.env.DEV,
})
```

The web URL is a config value. The page is a plain `.html`, so there is no `_spPageContextInfo`.
The debug page shows `WebFullUrl` from `contextinfo` to verify it.

## Requests

Formats:
- Reads: `application/json;odata=minimalmetadata`. Items come with `odata.etag`. Paging via `odata.nextLink`.
- Writes: `application/json;odata=verbose` with `__metadata.type`. The safe path on SharePoint 2016.

```ts
// lib/sharepoint/client.ts
const LIGHT = 'application/json;odata=minimalmetadata'
const VERBOSE = 'application/json;odata=verbose'

export type SpRequest = {
  method?: 'GET' | 'POST'
  body?: unknown // object → JSON. ArrayBuffer for files.
  headers?: Record<string, string>
  signal?: AbortSignal
  verbose?: boolean
}

export async function spFetch<T>(pathOrUrl: string, req: SpRequest = {}, retried = false): Promise<T> {
  const { webUrl, logger } = spConfig()
  const url = /^(https?:)?\//.test(pathOrUrl) ? pathOrUrl : `${webUrl}/_api/${pathOrUrl}`
  const method = req.method ?? 'GET'
  const type = req.verbose ? VERBOSE : LIGHT
  const headers: Record<string, string> = { Accept: type, ...req.headers }
  if (method === 'POST') {
    headers['Content-Type'] ??= type
    headers['X-RequestDigest'] = await getDigest()
  }
  const body = req.body === undefined || req.body instanceof ArrayBuffer ? req.body : JSON.stringify(req.body)

  const started = performance.now()
  const res = await fetch(url, { method, headers, body, signal: req.signal, credentials: 'same-origin' })
  logger?.request({ method: headers['X-HTTP-Method'] ?? method, url, status: res.status, ms: performance.now() - started })

  // Expired digest: refresh once and retry
  if (res.status === 403 && method === 'POST' && !retried) {
    clearDigest()
    return spFetch<T>(pathOrUrl, req, true)
  }
  if (!res.ok) throw await SpError.from(res)
  if (res.status === 204) return undefined as T
  return (await res.json()) as T
}
```

Request digest: cached, deduped while in flight, refreshed 60 s before it expires.

```ts
let digest: { value: string; expires: number } | undefined
let pending: Promise<string> | undefined

export function getDigest(): Promise<string> {
  if (digest && Date.now() < digest.expires) return Promise.resolve(digest.value)
  pending ??= fetch(`${spConfig().webUrl}/_api/contextinfo`, {
    method: 'POST',
    headers: { Accept: LIGHT },
    credentials: 'same-origin',
  })
    .then(async (res) => {
      if (!res.ok) throw await SpError.from(res)
      const info = (await res.json()) as { FormDigestValue: string; FormDigestTimeoutSeconds: number }
      digest = { value: info.FormDigestValue, expires: Date.now() + (info.FormDigestTimeoutSeconds - 60) * 1000 }
      return digest.value
    })
    .finally(() => { pending = undefined })
  return pending
}
export const clearDigest = () => { digest = undefined }
```

Errors: one class for both JSON formats. No constructor parameter properties (`erasableSyntaxOnly`).

```ts
export class SpError extends Error {
  status: number
  code?: string
  constructor(message: string, status: number, code?: string) {
    super(message)
    this.name = 'SpError'
    this.status = status
    this.code = code
  }
  static async from(res: Response) {
    const body = await res.json().catch(() => undefined)
    const e = body?.['odata.error'] ?? body?.error
    const message =
      res.status === 412 ? 'Someone else changed this item. Reload and try again.' : (e?.message?.value ?? res.statusText)
    return new SpError(message, res.status, e?.code)
  }
}
```

Logger: console only. Log method, status, duration, decoded URL. Never log request bodies with personal data.

## OData

```ts
// lib/sharepoint/odata.ts
export const lit = {
  str: (v: string) => `'${v.replace(/'/g, "''")}'`, // O'Brien → 'O''Brien'
  date: (isoUtc: string) => `datetime'${isoUtc}'`,
}

export const listPath = (title: string) => `web/lists/getbytitle(${lit.str(title)})`

export type ODataQuery = { select?: string[]; expand?: string[]; filter?: string; orderBy?: string; top?: number }

export function toQueryString(q: ODataQuery) {
  const p: string[] = []
  if (q.select?.length) p.push(`$select=${q.select.join(',')}`)
  if (q.expand?.length) p.push(`$expand=${q.expand.join(',')}`)
  if (q.filter) p.push(`$filter=${encodeURIComponent(q.filter)}`)
  if (q.orderBy) p.push(`$orderby=${encodeURIComponent(q.orderBy)}`)
  if (q.top) p.push(`$top=${q.top}`)
  return p.join('&')
}
```

`$select` and `$expand` come from the field descriptor. Never write them by hand.
Filter helpers (`eq`, `and`, `or`, `substringof`) are only needed for server-side mode later.

## Reading

Load a whole list. Ordered by ID (the default), so it stays safe above the 5,000 threshold.

```ts
// lib/sharepoint/list.ts
type Page = { value: unknown[]; 'odata.nextLink'?: string }

async function getAllRaw(list: ListDef, opts: { signal?: AbortSignal } = {}) {
  let next: string | undefined = `${listPath(list.title)}/items?${toQueryString({ ...selectFor(list), top: 5000 })}`
  const rows: unknown[] = []
  while (next) {
    const page: Page = await spFetch<Page>(next, { signal: opts.signal })
    rows.push(...page.value)
    next = page['odata.nextLink']
  }
  return rows
}
```

Parsing:
- Each item is parsed on its own (`safeParse`).
- `strictParse: true`: throw on the first failure (dev, tests).
- Otherwise: drop the bad item, `spLog.warn` with the Zod issues and the item id.

## Writing

```ts
export async function addItem(list: ListDef, values: object) {
  const res = await spFetch<{ d: { Id: number; __metadata: { etag: string } } }>(`${listPath(list.title)}/items`, {
    method: 'POST',
    verbose: true,
    body: { __metadata: { type: list.entityType }, ...toPayload(list, values) },
  })
  return { id: res.d.Id, etag: res.d.__metadata.etag }
}

export function updateItem(list: ListDef, id: number, values: object, etag: string) {
  return spFetch<void>(`${listPath(list.title)}/items(${id})`, {
    method: 'POST',
    verbose: true,
    headers: { 'X-HTTP-Method': 'MERGE', 'IF-MATCH': etag }, // MERGE, not PUT: PUT clears unset fields
    body: { __metadata: { type: list.entityType }, ...toPayload(list, values) },
  })
}

export function deleteItem(list: ListDef, id: number, etag = '*') {
  return spFetch<void>(`${listPath(list.title)}/items(${id})`, {
    method: 'POST',
    headers: { 'X-HTTP-Method': 'DELETE', 'IF-MATCH': etag },
  })
}
```

- `entityType` (`ListItemEntityTypeFullName`) is generated into the descriptor. No runtime lookup.
- Logo: first attachment. `POST items(id)/AttachmentFiles/add(FileName='logo.png')` with an `ArrayBuffer` body and `Content-Type: application/octet-stream`.
- `$batch` exists in SharePoint 2016, but it's not all-or-nothing. MVP uses parallel POSTs with `Promise.allSettled`.

## People

```ts
// lib/sharepoint/people.ts
export const currentUser = () =>
  spFetch<{ Id: number; Title: string; Email: string; LoginName: string }>('web/currentuser?$select=Id,Title,Email,LoginName')

export async function searchPeople(query: string, opts: { signal?: AbortSignal; max?: number } = {}) {
  const res = await spFetch<{ d: { ClientPeoplePickerSearchUser: string } }>(
    'SP.UI.ApplicationPages.ClientPeoplePickerWebServiceInterface.clientPeoplePickerSearchUser',
    {
      method: 'POST',
      verbose: true,
      signal: opts.signal,
      body: {
        queryParams: {
          __metadata: { type: 'SP.UI.ApplicationPages.ClientPeoplePickerQueryParameters' },
          QueryString: query,
          MaximumEntitySuggestions: opts.max ?? 10,
          PrincipalType: 1, // users
          PrincipalSource: 15, // all sources
          AllowEmailAddresses: false,
          AllowMultipleEntities: false,
          AllUrlZones: false,
        },
      },
    },
  )
  // The result is a JSON string inside JSON
  return pickerResultSchema.parse(JSON.parse(res.d.ClientPeoplePickerSearchUser))
}

// A picked person is not a site user yet. ensureUser returns the id a Person field stores.
export async function ensureUser(loginName: string) {
  const res = await spFetch<{ d: { Id: number; Title: string; Email: string } }>('web/ensureuser', {
    method: 'POST',
    verbose: true,
    body: { logonName: loginName }, // picker result `Key`
  })
  return { id: res.d.Id, title: res.d.Title, email: res.d.Email }
}
```

## Field descriptors

One descriptor per list. It is the single source for:
- `$select` and `$expand`
- the raw Zod schema and the transform to the domain object
- the write payload

Three shapes per list:

| Shape | Example | Where |
| --- | --- | --- |
| Raw SharePoint item | `{ Title, StartDate: '2026-10-19T22:00:00Z', Category: { Id, Title }, 'odata.etag' }` | inside the lib only |
| Domain object | `{ title, startDate: '2026-10-20', category: { id, title }, etag }` | everywhere in the app |
| Write payload | `{ __metadata, Title, StartDate, CategoryId, PresentersId: { results: [] } }` | inside the lib only |

Generated per list:

```ts
// src/features/conferences/model.gen.ts — generated, do not edit
import { defineList, f } from '@/lib/sharepoint'

export const conferencesList = defineList({
  title: 'Conferences',
  entityType: 'SP.Data.ConferencesListItem',
  fields: {
    id: f.id(),
    etag: f.etag(),
    title: f.text('Title'),
    description: f.note('Description'),
    startDate: f.date('StartDate'),
    endDate: f.date('EndDate'),
    category: f.lookup('Category'),
    organization: f.lookup('Organization'),
    country: f.choice('Country', ['Austria', 'Germany', 'Hungary']),
    location: f.text('Location'),
    presenters: f.userMulti('Presenters'),
    recurrenceFrequency: f.choice('RecurrenceFrequency', ['None', 'Weekly', 'Monthly', 'Yearly']),
    recurrenceInterval: f.number('RecurrenceInterval'),
    recurrenceCount: f.number('RecurrenceCount'),
    recurrenceUntil: f.date('RecurrenceUntil'),
    created: f.dateTime('Created', { readOnly: true }),
    author: f.user('Author', { readOnly: true }),
  },
})
```

`model.ts` (hand-written) builds the app type on top, e.g. groups the recurrence fields into `recurrence: { … }`.

How a field kind looks (shape, not the full file):

```ts
// lib/sharepoint/fields.ts
// verbose sends { results: [] }, JSON light sends [] — accept both
const spArray = <T extends z.ZodType>(item: T) =>
  z
    .union([z.array(item), z.object({ results: z.array(item) }).transform((v) => v.results)])
    .nullish()
    .transform((v) => v ?? [])

export const f = {
  text: (name: string) =>
    field({
      name,
      select: [name],
      read: z.string().nullish().transform((v) => v ?? ''),
      write: (v: string) => ({ [name]: v }),
    }),
  date: (name: string) =>
    field({
      name,
      select: [name],
      read: z.string().nullish().transform((v) => (v ? toDateOnly(v) : null)),
      write: (v: string | null) => ({ [name]: v ? fromDateOnly(v) : null }),
    }),
  lookup: (name: string) =>
    field({
      name,
      select: [`${name}/Id`, `${name}/Title`],
      expand: [name],
      // empty lookups come back in different shapes — check your fixtures
      read: z
        .object({ Id: z.number().nullish(), Title: z.string().nullish() })
        .nullish()
        .transform((v) => (v?.Id ? { id: v.Id, title: v.Title ?? '' } : null)),
      write: (id: number | null) => ({ [`${name}Id`]: id }),
    }),
  userMulti: (name: string) =>
    field({
      name,
      select: [`${name}/Id`, `${name}/Title`, `${name}/EMail`],
      expand: [name],
      read: spArray(z.object({ Id: z.number(), Title: z.string(), EMail: z.string().nullish() })).transform((users) =>
        users.map((u) => ({ id: u.Id, title: u.Title, email: u.EMail ?? '' })),
      ),
      write: (ids: number[]) => ({ [`${name}Id`]: { results: ids } }),
    }),
  // id, etag, note, number, choice, multiChoice, boolean, url, user, lookupMulti, dateTime, attachments…
}
```

Domain and write types are inferred from the descriptor. Never write them by hand.

## Cleaning rules

- Use internal names only (`StartDate`, `OData__x0020_…`). Never display names.
- Nulls: text → `''`, multi → `[]`, lookup/user → `null`, number/date → `null`.
- Unknown choice value (someone edited the choices in SharePoint): `null` + warning. Don't drop the item.
- Description: use a plain-text multi-line column. Rich text arrives wrapped in `<div class="ExternalClass…">` and needs sanitizing.
- Use `Id`, not `ID`. Both come back.

## Dates

All conference dates are date-only.
SharePoint stores a date-only value as midnight in the site timezone, sent as UTC.

```ts
// lib/sharepoint/dates.ts
import dayjs from 'dayjs'
import utc from 'dayjs/plugin/utc'
import timezone from 'dayjs/plugin/timezone'

dayjs.extend(utc)
dayjs.extend(timezone)

// '2026-10-19T22:00:00Z' → '2026-10-20' (Budapest, summer time)
export const toDateOnly = (spUtc: string) => dayjs.utc(spUtc).tz(spConfig().timeZone).format('YYYY-MM-DD')

// '2026-10-20' → '2026-10-19T22:00:00.000Z'
export const fromDateOnly = (date: string) => dayjs.tz(date, spConfig().timeZone).toISOString()

export const todayDateOnly = () => dayjs().tz(spConfig().timeZone).format('YYYY-MM-DD')
```

- In the app, dates are `'YYYY-MM-DD'` strings. They compare correctly as strings.
- Date math (recurrence, durations) uses `dayjs.utc(date)`. No clock-change surprises.
- The app imports a configured dayjs from `src/lib/dayjs.ts` (extends `utc` and `timezone`; extending twice is harmless).
- Mantine date pickers: convert at the component edge. The domain stays strings.

## Debug page

A route (`/debug`, gated). It runs same-origin with the user's Windows login, so it can read everything the user can.

Shows:
- All lists: title, item count, entity type (`web/lists?$filter=Hidden eq false`).
- Per list: fields (no `$select`, so choice/lookup/date details come back), the first 50 items as a table and raw JSON.
- A generic add/edit/delete form built from the fields.
- `contextinfo` (`WebFullUrl`, digest timeout), current user, site timezone (`web/RegionalSettings/TimeZone`).

Exports (browser download, no server):
- `sp-schema.json`: lists + fields + entity types. Input for codegen.
- `fixtures/<List>.json`: the first 20 raw items. Input for tests and MSW.

Checks panel: runs `lib/sharepoint/checks.ts` against live data. See [testing.md](./testing.md#after-codegen).

## Codegen

```
debug page → sp-schema.json → scripts/sp-codegen.ts → src/features/*/model.gen.ts
```

- Run: `node scripts/sp-codegen.ts sp-schema.json` (Node 24 runs TypeScript directly; else use `tsx`).
- A small map in the script: list title → feature folder, plus fields to skip (system fields).
- Generate only descriptors. No hooks, no queries. `queries.ts` is short and hand-written.
- Then run `tsc --noEmit` and the fixture check.

Type mapping (`TypeAsString`):

| SharePoint | Kind |
| --- | --- |
| Counter | `f.id()` |
| Text | `f.text` |
| Note | `f.note` |
| Number, Currency | `f.number` |
| Boolean | `f.boolean` |
| DateTime | `f.date` if `DisplayFormat` = 0 (date only), else `f.dateTime` |
| Choice / MultiChoice | `f.choice` / `f.multiChoice` with the `Choices` values |
| Lookup / LookupMulti | `f.lookup` / `f.lookupMulti` (target list from `LookupList` GUID) |
| User / UserMulti | `f.user` / `f.userMulti` |
| URL | `f.url` |
| Attachments | `f.attachments` |
| Computed | skipped |

## Lists

App-specific. Suggested columns.

| List | Columns | Settings |
| --- | --- | --- |
| Conferences | Title, Description (plain text), StartDate, EndDate (date only), Category (lookup), Organization (lookup), Country (choice), Location, Presenters (person, multi), RecurrenceFrequency (choice: None/Weekly/Monthly/Yearly), RecurrenceInterval, RecurrenceCount, RecurrenceUntil (date only), attachment = logo | index StartDate, Category |
| Tags | Title | enforce unique values (no duplicate tags) |
| ConferenceTags | Conference (lookup), Tag (lookup) | both lookups: enforce relationship, cascade delete |
| Registrations | Conference (lookup), OccurrenceDate (date only), Status (choice: Requested, Attending) | cascade delete; item-level permissions; user = Author |
| Follows | TargetType (choice: Category, Tag, Organization, Conference), TargetId (number) | item-level permissions; user = Author; like = TargetType Conference |
| Categories, Organizations | Title | |

- Item-level permissions: "Read all items", "Create items and edit items that were created by the user".
- Enforced unique values and enforced relationships need an indexed column. SharePoint creates it.

Recurrence: count and/or end date. Users register for one occurrence (`OccurrenceDate`).

```ts
// src/features/conferences/derive.ts
import { dayjs } from '@/lib/dayjs'

const UNIT = { Weekly: 'week', Monthly: 'month', Yearly: 'year' } as const
const MAX_OCCURRENCES = 100

export function expandOccurrences(c: Conference): Occurrence[] {
  const { frequency, interval, count, until } = c.recurrence
  if (frequency === 'None') return [{ start: c.startDate, end: c.endDate }]
  const lengthDays = dayjs.utc(c.endDate).diff(dayjs.utc(c.startDate), 'day')
  const limit = Math.min(count ?? MAX_OCCURRENCES, MAX_OCCURRENCES)
  const out: Occurrence[] = []
  for (let i = 0; i < limit; i++) {
    // Always from the original start: Jan 31 → Feb 28 → Mar 31 (no drift)
    const start = dayjs.utc(c.startDate).add(i * Math.max(interval ?? 1, 1), UNIT[frequency])
    const startStr = start.format('YYYY-MM-DD')
    if (until && startStr > until) break
    out.push({ start: startStr, end: start.add(lengthDays, 'day').format('YYYY-MM-DD') })
  }
  return out
}
```

## SharePoint 2016 limits

- `$skip` does not work on list items. Paging is `odata.nextLink` only.
- `$top` max is 5,000. The default page is 100.
- 5,000 item threshold: filtering or sorting on non-indexed columns fails on big lists.
- Max 12 lookup/person columns per query. Conferences uses 4 (Category, Organization, Presenters, Author).
- No `in` operator. Use `or` chains in chunks. Keep URLs under about 2,000 characters.
- Search API results lag behind (crawl). Don't use it for fresh data.
- Calendar-list recurrence: don't use. REST returns only the series master.
- Writes in `nometadata` without `__metadata` are not reliable on 2016. Use verbose writes.
- The request digest expires (`FormDigestTimeoutSeconds`, usually 1,800).
