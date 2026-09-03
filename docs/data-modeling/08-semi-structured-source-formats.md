# Source Formats: CSV, JSON, XML, and API Responses

Phase 0 profiling (`01-source-profiling-checklist.md`) assumes relational
tables by default. Source data often isn't relational yet — it arrives as
files or API payloads. This file extends Phase 0 with format-specific
first moves. The checks (nulls, format consistency, referential
integrity, PII, KPI discovery) still apply — this file covers how to get
to a queryable shape first, and what changes about the checks along the
way.

## General principle: land raw, profile, then structure (schema-on-read)

Consistent with ELT: land the file/API payload into Snowflake as close to
its original shape as possible before transforming. This preserves the
ability to re-derive the model if the initial structuring assumption
turns out wrong, and it's the natural point to do PII/format profiling on
the *actual* incoming shape rather than an assumed one.

- Semi-structured formats (JSON, XML, Avro, Parquet) land into a
  `VARIANT` column via `COPY INTO` — Snowflake stores the native
  structure and lets you query it with dot/bracket notation and
  `LATERAL FLATTEN` before deciding on a fixed relational shape
- Structured flat files (CSV/TSV) land into a typed staging table, but
  profile the raw text first (see below) before trusting inferred types

## CSV / TSV / delimited files

First-move checks before modeling:
- Confirm delimiter, quote character, escape character, and encoding
  (UTF-8 vs. Latin-1/Windows-1252 — a common source of silent corruption
  on names/addresses with accented characters)
- Confirm whether the file has a header row, and whether column order is
  guaranteed stable across extracts (don't assume — some source exports
  reorder columns when new fields are added)
- Check for embedded delimiters/newlines inside quoted fields — these
  break naive line-counting (`wc -l` over-counts) and naive splitting
- Type-infer cautiously: a column that's "numeric" in one extract can
  become alphanumeric in a later one (e.g. an ID field that changes
  format) — if profiling multiple historical extracts, check type
  stability across them, not just the current file
- Watch for leading zeros lost on numeric-looking codes (e.g. account
  numbers, postal codes) if any tool in the pipeline auto-types them as
  numbers — flag this as a format-consistency risk, not just a null issue

```sql
COPY INTO stg.raw_customer_csv
FROM @my_stage/customer_extract.csv
FILE_FORMAT = (TYPE = CSV, FIELD_OPTIONALLY_ENCLOSED_BY = '"',
               SKIP_HEADER = 1, ENCODING = 'UTF-8');
```

## JSON (files or API payloads)

First-move checks before modeling:
- **Schema drift**: JSON is schema-on-read — different records in the
  same feed can have different keys present, nested differently, or
  different types for the same key across records/time. Profile a
  representative sample across multiple extracts/time periods, not just
  one payload, and explicitly check for drift before assuming a fixed
  schema
- **Nesting depth and arrays**: identify which fields are scalar, which
  are nested objects, and which are arrays (arrays need `LATERAL
  FLATTEN` to become rows — decide the target grain when flattening, same
  discipline as fact-table grain-setting)
- **Null vs. absent-key**: a JSON field can be explicitly `null` or
  simply missing from the object entirely — these are different in
  `VARIANT` handling and can mean different things (explicit null =
  known-empty; absent key = not captured / older schema version).
  Distinguish them in the completeness analysis rather than treating both
  as "null"

```sql
CREATE TABLE stg.raw_orders_json (raw_payload VARIANT, _loaded_at TIMESTAMP_NTZ);
COPY INTO stg.raw_orders_json (raw_payload)
FROM @my_stage/orders.json
FILE_FORMAT = (TYPE = JSON);

-- Inspect distinct top-level key sets across records to detect schema drift
SELECT ARRAY_AGG(DISTINCT key) WITHIN GROUP (ORDER BY key)
FROM stg.raw_orders_json, LATERAL FLATTEN(input => raw_payload);

-- Flatten a nested array to its natural grain
SELECT raw_payload:order_id::STRING AS order_id,
       f.value:sku::STRING AS sku,
       f.value:qty::NUMBER AS quantity
FROM stg.raw_orders_json,
     LATERAL FLATTEN(input => raw_payload:line_items) f;
```

## XML

First-move checks before modeling:
- Distinguish data carried as **elements** vs. **attributes** — both are
  common in the same document and need different extraction paths
- Check for and note namespaces (`xmlns`) — they affect path expressions
  and are easy to silently drop during extraction, which can cause
  fields to appear "missing" when they're actually just namespace-scoped
- Confirm whether the XML is one-record-per-file or many-records-per-file
  (a batch document with repeating elements) — this determines the
  flattening/grain approach, same as JSON arrays

Snowflake approach: land as `VARIANT` via the `TYPE = XML` file format,
then use `GET_PATH`/`XMLGET` style extraction, or pre-convert to JSON
upstream if the pipeline tooling (NiFi, SnapLogic) supports it more
cleanly than native SQL XML handling.

## API responses (REST/SOAP)

API sources add operational profiling questions on top of the payload
questions already covered under JSON/XML above — ask, don't assume:

- **Pagination**: does the API paginate, and is the full profiled sample
  actually the complete dataset or just page 1? Confirm pagination
  handling before drawing completeness/null conclusions from a partial
  pull
- **Incremental extraction contract**: does the API support a
  since/watermark parameter (preferred), or does it only return full
  snapshots — this determines the incremental load design (see
  `02-dimensional-modeling-guide.md` / Phase 4 load-type documentation),
  not just the profiling step
- **Rate limits and batching**: relevant to ELT design, not modeling
  directly, but worth capturing in the profiling report since it affects
  how fresh the landed data can realistically be
- **Schema versioning**: does the API contract version its response
  schema? If fields can be added/removed/renamed across API versions,
  treat this the same as JSON schema drift above, and confirm with the
  user how version transitions should be handled (backfill, map old→new
  field names, accept a schema change date boundary)
- **Error/partial-response records**: check whether the API embeds
  per-record error states in an otherwise-200 response (common in batch
  API patterns) — these need to be excluded from completeness/null
  analysis or handled as their own category, not treated as valid nulls

## PII screening in semi-structured data

PII can be nested arbitrarily deep in JSON/XML (e.g. inside a
`metadata.contact.email` path, or inside a free-text `notes` element).
Column-name-based screening (as in `01-source-profiling-checklist.md`)
isn't sufficient here — walk nested paths and sample values at each leaf,
not just top-level keys, before concluding a payload is PII-free.

## What doesn't change

Once semi-structured data is flattened into a typed staging table, every
other Phase 0 check — null/completeness analysis, referential integrity,
PII classification, KPI discovery — applies exactly as documented in
`01-source-profiling-checklist.md`. The format-specific work above is
purely about getting from "file/API payload" to "queryable typed table"
without losing information or silently mis-assuming structure along the
way.
