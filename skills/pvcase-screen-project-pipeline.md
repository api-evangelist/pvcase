---
name: pvcase-screen-project-pipeline
description: >-
  Screen a PVcase Prospect solar development pipeline — list the teams an API token can
  reach, page through projects with the AG Grid query body, and pull the parcel, substation
  and transmission-line assets attached to a project.
api: PVcase Prospect API (Anderson Optimization API)
generated: '2026-08-26'
method: generated
source: openapi/pvcase-anderson-optimization-openapi.json
operations:
  - GET /api/teams
  - POST /api/projects/query
  - POST /api/teams/{teamId}/projects/query
  - GET /api/teams/{teamId}/projects/{projectId}
  - POST /api/assets/query/project
  - POST /api/assets/query/column
---

# Screen a PVcase Prospect project pipeline

> The published OpenAPI declares **no `operationId` on any of its 48 operations**, so every
> step below is identified by METHOD + path. Do not invent operationIds when calling.

## Before you start

- **Base URL.** The spec declares `https://core-v1.carbon.prod.andersonopt.com` and the docs'
  Quick Start uses `https://api.andersonopt.com`. As probed on 2026-08-26 **neither hostname
  resolves**. Confirm the live base URL with PVcase support before running this skill; do
  not assume either published value works.
- **Auth.** HTTP Basic. Username is the AO username, password is the API token. Mint or
  rotate a token with `GET /auth/token` (Basic auth with the existing token), or from the
  user profile in the Prospect UI. Treat the token like a password.
- **Access.** This API is enterprise, sales-gated and documented as a "Future Offering".
  There is no self-serve signup.

## Step 1 — find the teams you can reach

`GET /api/teams`

Returns every team the token's user belongs to. A team is the unit of access control and
scopes everything below. Keep the `teamId` values.

## Step 2 — page through projects

`POST /api/projects/query` (all teams) or `POST /api/teams/{teamId}/projects/query`
(one team).

The body is an **AG Grid server-side row model** request, not query-string paging:

- `startRow` / `endRow` — the slice to return
- `sortModel` — `[{ "colId": "...", "sort": "asc" | "desc" }]`
- `filterModel` — the filter tree
- `rowGroupCols` / `valueCols` / `groupKeys` — grouping and aggregation
- Response: `{ "rowData": [...], "totalCount": <int> }`

Loop by advancing `startRow`/`endRow` until you have consumed `totalCount`. There is **no
cursor and no next-link**, and no maximum page size is documented — start conservatively.

To narrow the pipeline first, call `POST /api/projects/query/recent` or
`POST /api/projects/query/favorite`, which return the user's recent and favourited projects.

## Step 3 — build filter pickers from real values

`POST /api/projects/query/column` and `POST /api/assets/query/column` return the **distinct
values** of a column. Use these to build a `filterModel` from values that actually exist
rather than guessing enum members — the spec's 252 component schemas are anonymous
(`def-0` … `def-251`) and carry no titles, so you cannot read the vocabulary out of it.

## Step 4 — read one project

`GET /api/teams/{teamId}/projects/{projectId}`

A project carries an open-ended `parameter` object and an `assets` field. The `parameter`
keys are **not in the spec** — they are documented in an external Google Sheet linked from
the Projects reference page. Treat unknown keys as pass-through data.

## Step 5 — pull the assets behind a project

`POST /api/assets/query/project` — assets filtered to a project. Assets are land parcels,
substations, transmission lines and points of interconnection. As with projects, the
`parameter` field varies by asset subtype and is documented in a separate spreadsheet.

For map rendering rather than tabular reads, use the XYZ tile operations
`GET /api/teams/{teamId}/assets/{z}/{x}/{y}` with `filterType=asset:network:line` or
`filterType=asset:network:substation`.

## Error handling

Errors are **bare JSON strings**, not RFC 9457 problem documents. You can only branch on
the HTTP status:

| Status | Meaning | What to do |
|---|---|---|
| 401 | `"Requires Authentication"` | Re-check Basic credentials; mint a new token at `GET /auth/token` |
| 404 | Resource not found | Verify `teamId` / `cid` / `projectId` / `assetId` |
| 500 | `"Internal server error"` | Retry with backoff, then contact support@pvcase.com |

**No 400, 403, 409, 422 or 429 is documented on any operation.** In particular there is no
documented rate limit and no `Retry-After`, so treat repeated 500s as possible backpressure
and back off exponentially.

## Do not

- Do not call the write operations from this skill. Creates have **no idempotency key**, so
  a retry duplicates; updates and deletes have **no documented reversal, restore or
  retention window**. See `conventions/pvcase-conventions.yml`.
