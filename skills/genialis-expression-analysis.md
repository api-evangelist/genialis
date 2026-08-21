---
name: genialis-expression-analysis
description: >-
  Pull an expression matrix for a set of Genialis samples, read the differential
  expression results attached to a collection, and resolve gene identifiers through
  the platform knowledge base.
api: Genialis Expressions API
base_url: https://app.genialis.com
spec: openapi/genialis-base-openapi.yaml
generated: '2026-08-21'
method: generated
source: >-
  Grounded in operationIds verified against openapi/genialis-base-openapi.yaml on
  2026-08-21.
operations:
  - api__modules_sample_table_create
  - api__modules_differential_expression_list_list
  - api__modules_differential_expression_geneset_search_list
  - api__modules_gene_list_list_list
  - api__modules_gene_list_list_by_ids_list
  - api__modules_species_collection_list
  - api_annotation_field_list
  - api_annotation_value_list
---

# Expression analysis on Genialis Expressions

The `/api/_modules/` namespace is the analysis surface the Expressions frontend uses. It is in the
published contract and is reachable the same way every other path is.

## Step 1 — get the expression / metadata table

`api__modules_sample_table_create` — `POST /api/_modules/sample/table`

This is a POST because the sample set and the requested columns go in the body, not because it
mutates anything — it is a read expressed as a POST. Nothing is created.

The published contract declares **no request schema and no parameters** for this operation, so build
the body from what the platform documents rather than guessing: see
<https://docs.genialis.com/resdk/> and the ReSDK `Collection.samples` / tables interface, which is
the supported way to reach this data. If you are not certain of the body shape, use `resdk` rather
than hand-rolling the request.

## Step 2 — read differential expression results

`api__modules_differential_expression_list_list` — `GET /api/_modules/differential_expression/list`

```
GET https://app.genialis.com/api/_modules/differential_expression/list?collection=<collection_id>&limit=25
```

Filters mirror the data filters: `collection`, `collection__slug`, `build`, `basket`, `created__gte`.

To search results by gene set:
`api__modules_differential_expression_geneset_search_list` —
`GET /api/_modules/differential_expression/geneset_search`

**Read `genialis-explore-collections.md` step 4 first.** A differential expression result is only
interpretable against the `Relation` partitions that defined the comparison.

## Step 3 — resolve genes

`api__modules_gene_list_list_list` — `GET /api/_modules/gene_list/list?data=<data_id>&limit=100`
(supports `limit`, `offset`, `ordering`).

By identifier: `api__modules_gene_list_list_by_ids_list` —
`GET /api/_modules/gene_list/list_by_ids`.

Species context: `api__modules_species_collection_list` —
`GET /api/_modules/species/collection`.

## Step 4 — attach sample metadata

Expression values mean little without the sample annotations.

- `api_annotation_field_list` — `GET /api/annotation_field?collection=<collection_id>` returns the
  fields defined for that collection (with their `group`, `type`, `label`, `vocabulary`).
- `api_annotation_value_list` — `GET /api/annotation_value?entity=<sample_id>` returns the values
  for one sample; filter by `field` or `entity__collection` to pull a whole collection's metadata.

## Rules

- Everything above is a read. Do not call `/api/data/{id}/restart`, any `duplicate`, or any
  `bulk_delete` from this skill — those have no idempotency key, no dry-run and no documented undo.
- Page everything: `{count, next, previous, results}`.
- Check `content-type: application/json`. A 200 with `text/html` is the SPA shell and means the path
  was wrong.
