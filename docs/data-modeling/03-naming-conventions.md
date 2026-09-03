# Snowflake Naming Conventions

Apply these consistently in Phase 3 (Physical Data Model). If the user's
organization already has an established convention that conflicts with
this, **ask which takes precedence** — don't silently override an
existing house style. Treat this file as the default when none exists.

## General rules

- All identifiers: lowercase with underscores (`snake_case`), even though
  Snowflake supports quoted mixed-case — unquoted lowercase avoids
  case-sensitivity foot-guns across tools (SnapLogic, dbt, BI tools)
- No reserved words as identifiers (`ORDER`, `DATE`, `USER`, etc.) —
  prefix/suffix instead (`order_date`, not `order`)
- No spaces, no special characters beyond underscore
- Keep names descriptive but concise; avoid unexplained abbreviations
  unless they're an established business glossary term (confirm with user)

## Database / schema layering (ELT pattern)

Reflect the ELT flow (land raw → transform in-warehouse → serve) in schema
naming so lineage is visible from the object name alone:

| Layer | Schema naming pattern | Purpose |
|---|---|---|
| Raw / landing | `<db>.raw_<source_system>` or `<db>.stg_raw` | Untransformed, as-landed data from source (SnapLogic/NiFi drop zone) |
| Staging | `<db>.stg` | Cleansed, typed, deduplicated; still source-grain, pre-conformance |
| Conformed / dimensional | `<db>.dwh` or `<db>.core` | Facts and dimensions — the star schema layer |
| Marts | `<db>.mart_<subject_area>` | Subject-area-specific views/tables built on the conformed layer (e.g. `mart_risk`, `mart_sales`) |
| Semantic/BI | `<db>.bi` or handled in the BI tool's semantic layer | Presentation-friendly views, renamed for business consumption if needed |

## Table naming

| Object type | Pattern | Example |
|---|---|---|
| Fact table | `fact_<subject>` | `fact_transactions`, `fact_account_balance_daily` |
| Dimension table | `dim_<entity>` | `dim_customer`, `dim_product`, `dim_date` |
| Bridge/associative table (for M:N) | `bridge_<entity1>_<entity2>` | `bridge_account_holder` |
| Staging table | `stg_<source_system>_<entity>` | `stg_core_banking_customer` |
| Raw/landing table | `raw_<source_system>_<entity>` | `raw_snaplogic_orders` |
| View | Same as underlying pattern + purpose suffix if needed | `vw_fact_transactions_daily` |

Include grain or cadence in the fact name when it isn't obvious, e.g.
`fact_account_balance_daily` vs. `fact_account_balance_monthly` if both
exist — this prevents accidental grain-mismatched joins.

## Column naming

| Element | Pattern | Example |
|---|---|---|
| Surrogate key (dimension PK) | `<entity>_sk` or `<entity>_key` | `customer_sk` |
| Natural/business key (source system ID) | `<entity>_id` or `<source>_<entity>_id` | `customer_id`, `core_banking_customer_id` |
| Foreign key in fact table | `<dimension>_sk` matching the referenced dimension's surrogate key | `customer_sk`, `product_sk`, `date_sk` |
| Measure columns | Descriptive noun, unit implied by a comment or explicit suffix if ambiguous | `sales_amount`, `quantity_sold`, `balance_amount_usd` |
| Date/time columns | `<event>_date` or `<event>_timestamp`/`_ts` | `transaction_date`, `created_ts` |
| SCD Type 2 tracking columns | `valid_from`, `valid_to`, `is_current` (boolean) | — |
| Flags/booleans | `is_<condition>` or `has_<condition>` | `is_active`, `has_kyc_complete` |
| Audit/ELT metadata columns | `_loaded_at`, `_source_system`, `_batch_id`, `_row_hash` (prefix with underscore to visually separate from business columns) | `_loaded_at`, `_source_system` |

## Keys and constraints

Snowflake does not enforce PK/FK/UNIQUE constraints for performance
(they're informational only), but declare them anyway — they document
intent, and some tools (dbt, BI semantic layers, ER diagram generators)
read them for relationship inference.

```sql
CREATE TABLE dwh.dim_customer (
    customer_sk        NUMBER      NOT NULL,   -- surrogate key
    customer_id         VARCHAR(50) NOT NULL,   -- natural/business key from source
    customer_name        VARCHAR(200),
    segment              VARCHAR(50),
    valid_from           DATE        NOT NULL,
    valid_to              DATE        NOT NULL,
    is_current            BOOLEAN     NOT NULL,
    _source_system        VARCHAR(50),
    _loaded_at             TIMESTAMP_NTZ,
    CONSTRAINT pk_dim_customer PRIMARY KEY (customer_sk)
);

CREATE TABLE dwh.fact_transactions (
    transaction_sk      NUMBER      NOT NULL,
    customer_sk          NUMBER      NOT NULL,
    product_sk            NUMBER      NOT NULL,
    date_sk                NUMBER      NOT NULL,
    transaction_amount     NUMBER(18,2),
    quantity                NUMBER,
    _source_system         VARCHAR(50),
    _loaded_at              TIMESTAMP_NTZ,
    CONSTRAINT pk_fact_transactions PRIMARY KEY (transaction_sk),
    CONSTRAINT fk_fact_transactions_customer FOREIGN KEY (customer_sk) REFERENCES dwh.dim_customer(customer_sk),
    CONSTRAINT fk_fact_transactions_product FOREIGN KEY (product_sk) REFERENCES dwh.dim_product(product_sk),
    CONSTRAINT fk_fact_transactions_date FOREIGN KEY (date_sk) REFERENCES dwh.dim_date(date_sk)
);
```

## Performance objects

- Clustering keys: name/document the rationale in a comment; typically the
  most frequently filtered/joined column(s) on large fact tables (e.g.
  `transaction_date`)
- Materialized views / dynamic tables: prefix with `mv_` / follow the same
  entity naming as their base object

## PII/masking policy naming

- Masking policy objects: `mask_<data_class>` e.g. `mask_pii_email`,
  `mask_pii_account_number`
- Row access policies: `rap_<purpose>` e.g. `rap_region_restrict`
- Tag names for classification (Snowflake object tagging): `pii_class`,
  `data_sensitivity` with standard tag values from the taxonomy in
  `01-source-profiling-checklist.md`
