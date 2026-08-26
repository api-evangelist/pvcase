---
name: pvcase-inventory-grid-assets
description: >-
  Inventory grid infrastructure across a whole PVcase Prospect company — substations,
  transmission lines and land parcels — using the company-scoped query and vector-tile
  operations, without touching any write surface.
api: PVcase Prospect API (Anderson Optimization API)
generated: '2026-08-26'
method: generated
source: openapi/pvcase-anderson-optimization-openapi.json
operations:
  - GET /api/companies/{cid}
  - POST /api/companies/{cid}/assets/query
  - POST /api/companies/{cid}/assets/query/project
  - POST /api/companies/{cid}/projects/query
  - GET /api/companies/{cid}/assets/{z}/{x}/{y}
  - POST /api/assets/query/column
---

# Inventory grid assets across a PVcase Prospect company

> No `operationId` exists in the published spec. Call by METHOD + path.
> Read-only skill: every operation below is a GET or a query POST. Nothing here mutates state.

## Scope

`Company` is the top-level tenant and sits **above** teams. Company-scoped queries span every
team beneath the company, which is what makes this the right entry point for a
portfolio-wide inventory rather than a per-team read. Note that `Company` is **read-only** in
this contract — no create, update or delete operations are exposed for it.

## Step 1 — resolve the company

`GET /api/companies/{cid}`

You need the `cid`. It is not discoverable from `GET /api/teams`; obtain it from PVcase or
from the Prospect UI.

## Step 2 — inventory assets company-wide

`POST /api/companies/{cid}/assets/query`

Same AG Grid body as everywhere else — `startRow`, `endRow`, `sortModel`, `filterModel`,
`rowGroupCols`, `valueCols`. The response is `{ rowData, totalCount }`.

Use `rowGroupCols` + `valueCols` with an `aggFunc` to get counts by asset subtype server-side
instead of paging the whole set and counting client-side.

Related narrowings:

- `POST /api/companies/{cid}/assets/query/project` — assets grouped by project
- `POST /api/companies/{cid}/assets/query/project/recent` — recently touched projects only
- `POST /api/companies/{cid}/assets/query/project/favorite` — favourited projects only
- `POST /api/companies/{cid}/projects/query` — the project layer of the same inventory

## Step 3 — discover the asset vocabulary before filtering

`POST /api/assets/query/column`

Returns distinct values for a column. Run this **first** for any column you intend to filter
on. The 252 component schemas in the spec are anonymous `def-N` objects with no titles, and
the `parameter` field keys are documented only in an external Google Sheet — so the distinct
-values endpoint is the only machine-readable source of the real vocabulary.

Asset subtypes observed in PVcase's own client-side layer filters:

- `asset:network:line` — transmission lines
- `asset:network:substation` — substations
- land parcels and points of interconnection (POI) also appear as asset types

## Step 4 — render the inventory on a map

`GET /api/companies/{cid}/assets/{z}/{x}/{y}` (and the `/projects/` equivalent)

Standard XYZ tile addressing, with an optional `filterType` query parameter. A `POST` form of
the same path exists which accepts a query body, so you can render a **filtered subset** as
tiles rather than fetching everything and filtering client-side.

Caveat: the spec declares the tile response as `application/json`, but the PVcase Prospect
client decodes these as Mapbox Vector Tiles. Inspect the actual content-type before parsing.

## Error handling

Only 401 / 404 / 500 are documented, as bare JSON strings. No 429 and no `Retry-After`
exists, so rate-limit exhaustion is indistinguishable from a server fault — back off on
repeated 500s. See `errors/pvcase-problem-types.yml`.
