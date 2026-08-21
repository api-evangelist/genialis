---
name: genialis-variant-lookup
description: >-
  Look up genomic variants and per-sample variant calls on Genialis Expressions,
  including dbSNP and ClinVar annotation and the clinical significance vocabulary.
api: Genialis Expressions API
base_url: https://app.genialis.com
spec: openapi/genialis-base-openapi.yaml
generated: '2026-08-21'
method: generated
source: >-
  Grounded in operationIds and component schemas verified against
  openapi/genialis-base-openapi.yaml on 2026-08-21.
operations:
  - api_variant_list
  - api_variant_calls_list
  - api_variant_annotations_list
  - api_variant_experiment_list
---

# Variant lookup on Genialis Expressions

The variant surface is a direct projection of the **VCF v4.x** call model, so if you already speak
VCF you can map straight onto it with no translation layer.

| VCF | Genialis field | Resource |
|---|---|---|
| CHROM | `chromosome` | `/api/variant` |
| POS | `position` | `/api/variant` |
| REF | `reference` | `/api/variant` |
| ALT | `alternative` | `/api/variant` |
| QUAL | `quality` | `/api/variant_calls` |
| FILTER | `filter` | `/api/variant_calls` |
| FORMAT/GT | `genotype` | `/api/variant_calls` |
| FORMAT/GQ | `genotype_quality` | `/api/variant_calls` |
| FORMAT/DP | `depth` | `/api/variant_calls` |
| FORMAT/AD | `alternative_allele_depth` | `/api/variant_calls` |

`genome_assembly` and `species` are on `/api/variant`. **Always filter on `genome_assembly`** — a
position is meaningless without the build it was called against.

## Step 1 — find the variant

`api_variant_list` — `GET /api/variant`

```
GET https://app.genialis.com/api/variant?chromosome=chr12&position=25245350&genome_assembly=<build>&limit=25
```

Filters in the contract cover `chromosome`, `position` (with `__gt/__gte/__lt/__lte/__in`),
`reference`, `alternative`, `species`, `genome_assembly`, and — reaching through to the annotation —
`annotation__clinical_diagnosis`, `annotation__clinical_significance`, `annotation__dbsnp_id`,
`annotation__clinvar_id`. Each supports the Django lookup suffixes (`__in`, `__icontains`,
`__isnull`, `__startswith`).

**There is no single-variant GET.** `/api/variant/{id}` exists in the contract but exposes only
`api_variant_update`, `api_variant_partial_update` and `api_variant_destroy` — PUT, PATCH and DELETE,
no GET. To fetch one variant, filter the list: `GET /api/variant?id=<id>`. Do not construct
`GET /api/variant/{id}`; on this host it will return the SPA shell with a 200.

## Step 2 — read the annotation

`api_variant_annotations_list` — `GET /api/variant_annotations?variant_id=<id>`

Returns `type`, `clinical_diagnosis`, `clinical_significance`, `dbsnp_id`, `clinvar_id` and
`transcripts`. `dbsnp_id` and `clinvar_id` are the NCBI identifier schemes — use them to join out to
dbSNP/ClinVar rather than re-deriving significance yourself.

## Step 3 — read per-sample calls

`api_variant_calls_list` — `GET /api/variant_calls`

```
GET https://app.genialis.com/api/variant_calls?variant=<variant_id>&limit=50
```

Filter by `sample`, `variant`, `experiment`, `data`, `data__collection`, and by call quality:
`quality__gte`, `genotype_quality__gte`, `depth__gte`, `alternative_allele_depth__gte`, `filter`.

Apply a quality floor before you report anything. A call with low `depth` or a non-PASS `filter` is
not a finding.

## Step 4 — provenance

`api_variant_experiment_list` — `GET /api/variant_experiment` gives the `timestamp`,
`contributor` and `variant_data_source` for a calling run. `VariantCall.data` links back to the
`/api/data/{id}` object the calls came out of, which links to the `process` that produced them —
that is the full provenance chain from a call to the pipeline that made it.

## Rules

- Read-only. Genuine write operations exist on these resources (`api_variant_create`,
  `api_variant_calls_destroy`, …); this skill does not use them and neither should an agent without
  explicit human instruction — there is no idempotency key, no dry-run and no documented undo.
- Anonymous access reaches public data only; private cohorts need a session. See
  `genialis-authenticate.md`.
- Page with `limit`/`offset`; check `content-type: application/json` on every response.
