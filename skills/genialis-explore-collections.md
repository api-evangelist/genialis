---
name: genialis-explore-collections
description: >-
  Browse the Genialis Expressions catalog — find collections, list the samples in a
  collection, and open a data object with its process status — using only anonymous
  reads against the public REST API.
api: Genialis Expressions API
base_url: https://app.genialis.com
spec: openapi/genialis-base-openapi.yaml
generated: '2026-08-21'
method: generated
source: >-
  Grounded in operationIds verified against openapi/genialis-base-openapi.yaml on
  2026-08-21. Live behaviour confirmed by anonymous GET requests.
operations:
  - api_collection_list
  - api_collection_retrieve
  - api_sample_list
  - api_sample_retrieve
  - api_data_list
  - api_data_retrieve
  - api_relation_list
---

# Explore Genialis Expressions collections

## Before you start

- Base URL is `https://app.genialis.com`. Paths are server-root relative. Resource paths live under
  `/api/`, but `/about/*`, `/rest-auth/*`, `/saml-auth/*` and `/health_check` do **not**.
- **Assert `content-type: application/json` on every response.** This host is a single-page-app
  catch-all: an unmatched path returns HTTP **200** with a 1377-byte HTML shell, not a 404. A 200 is
  not proof the call worked.
- Public/community data is readable **anonymously**. Verified 2026-08-21:
  `GET /api/data?limit=1` -> 200, `count: 29123`. Private objects require a session
  (`cookieAuth`, cookie `sessionid`) — see `genialis-authenticate.md`.
- This whole skill is read-only. Nothing here mutates state.

## Step 1 — list collections

`api_collection_list` — `GET /api/collection`

```
GET https://app.genialis.com/api/collection?limit=25&ordering=-created
```

Response is DRF limit/offset: `{count, next, previous, results}`. Page with `limit` and `offset`, or
follow the absolute `next` URL. Useful filters exposed by the contract: `name__icontains`,
`description__icontains`, `contributor`, `created__gte`, `created__lte`, `slug`, `tags`.

## Step 2 — list the samples in a collection

`api_sample_list` — `GET /api/sample`

```
GET https://app.genialis.com/api/sample?collection=<collection_id>&limit=50
```

A "sample" is a Resolwe **Entity**. Filters include `collection`, `collection__slug`,
`collection__name__icontains`, `name__icontains`, `annotations`, and `basket`.

Fetch one with `api_sample_retrieve` — `GET /api/sample/{id}`.

## Step 3 — list the data objects

`api_data_list` — `GET /api/data`

```
GET https://app.genialis.com/api/data?collection=<collection_id>&limit=50
```

Filter by `entity` (the sample), `process`, `status`, `build`, `type`, `created__gte`. Retrieve one
with `api_data_retrieve` — `GET /api/data/{id}`. The returned object carries `status`
(see the `DataStatus` schema), `process`, `collection`, `entity`, `descriptor_schema`, `created`,
`started`, `finished` and `checksum`.

## Step 4 — read the grouping structure

`api_relation_list` — `GET /api/relation?collection=<collection_id>`

Relations are the typed groupings inside a collection — series, replicate groups, case/control
comparisons — with their `partitions`. Read these before interpreting any differential-expression
result, because they define what was compared against what.

## Sorting and failure

- Sort with `ordering=<field>` or `ordering=-<field>`.
- An unsupported field returns **400** with `{"error": "Ordering by field 'x' is not supported."}`.
  That is the actual envelope — this API does **not** use RFC 9457 `application/problem+json`, and
  its published spec documents no error responses at all, so handle failures defensively.
- There is no rate-limit header and no `Retry-After`. Pace yourself client-side; nothing will tell
  you that you are going too fast.
