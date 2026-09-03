# Dimensional Modeling Guide

Reference for Phase 2 (Logical Data Model) decisions. Every decision below
is a business decision — confirm with the user rather than defaulting.

## Setting the grain (do this first, always)

The grain is a single declarative sentence describing exactly what one row
in the fact table represents, e.g. "one row per line item per order" or
"one row per account per day." Every measure and every dimensional FK in
that fact table must be true at that grain.

Process:
1. Propose a grain in plain English based on the source data's natural
   transaction unit
2. State it back to the user explicitly: "I'm proposing the grain of
   `fact_orders` as one row per order line item — confirm?"
3. Only after confirmation, attach measures and dimension FKs

Get the grain wrong and everything downstream (additivity, SCD join
timing, KPI accuracy) breaks silently. If the source data mixes grains
(e.g. some rows are order-level, some are line-item-level), stop and flag
it — this is a data quality issue, not a modeling nuance to paper over.

## Facts and dimensions

- **Fact table**: quantitative, measurable data (sales amount, quantity,
  balance, count). Numeric, generally aggregated in reports (sum, avg,
  etc.), linked to dimensions via foreign keys.
- **Dimension table**: descriptive, categorical context for facts
  (customer, product, date, branch, account type). Non-numeric, used to
  group/filter/slice facts.

## Additivity of measures

Every measure in a fact table must be classified. This determines whether
it can be validly summed by BI tools/dashboards across a given dimension.

| Type | Can be summed across | Examples | Modeling implication |
|---|---|---|---|
| **Additive** | All dimensions | Revenue, quantity sold, transaction count | Safe to expose as a simple `SUM()` measure everywhere |
| **Semi-additive** | Some dimensions, but **not time** | Account balances, inventory levels, headcount | Must define the correct time semantics (e.g. "balance as of period end", not summed across days) — document explicitly which aggregation (last, average, max) is valid over time |
| **Non-additive** | No dimension | Ratios, percentages, averages, margins | Must be recalculated from underlying additive components at the target grain, never summed directly — document the exact recalculation formula |

For every measure, ask: "if a dashboard sums this column across [time /
product / region], does the result mean anything?" If the answer is ever
"no", classify it accordingly and document the correct aggregation
approach in the transformation spec (Phase 4). Getting this wrong is the
single most common cause of misleading dashboards.

## Slowly Changing Dimensions (SCD)

A dimension attribute can change over time (customer address, product
category, relationship manager assignment, KYC status, etc.). Decide the
SCD type **per attribute**, not just per table — a `dim_customer` table
commonly mixes Type 1 attributes (e.g. email, phone) with Type 2
attributes (e.g. address, segment, risk rating) with Type 0 attributes
(e.g. original account-opening channel, date of birth).

| Type | Method | History | Use when | Tradeoff |
|---|---|---|---|---|
| **Type 0** | Never overwritten after initial insert | Original value retained permanently | Immutable-by-design or audit/regulatory fields (e.g. original acquisition channel, account opening date) | No ability to reflect real change — that's the point |
| **Type 1** | Overwrite in place | None preserved | Correcting errors, non-critical attributes where only current state matters (e.g. fixing a name typo) | Fast, minimal storage, but breaks historical/point-in-time reporting on that attribute |
| **Type 2** | New row per change, with effective-dated validity (`valid_from`/`valid_to` or `effective_date`/`end_date`, plus a `current_flag`) | Full history preserved | Attribute changes must be tracked for accurate historical reporting (e.g. customer address, product category, relationship manager, risk segment) | Higher storage, more ETL/ELT complexity (surrogate key generation, date handling), but this is the default for anything analytically significant that changes |
| **Type 3** | Add `current_value` + `previous_value` columns | Only most recent prior state | Only the last change matters, full lineage not required (e.g. product recategorization where only current vs. prior category is queried) | Simpler and cheaper than Type 2, but cannot support full trend analysis |
| **Hybrid** | Mix of the above per attribute within one dimension table | Varies by attribute | Common in practice — most real dimension tables are hybrid | Document per-attribute, not per-table |

For Type 2, always confirm with the user:
- Surrogate key strategy (sequence, `HASH()`, `UUID`)
- `valid_from` / `valid_to` convention, and what value represents "open"
  (commonly `9999-12-31` as seen in source systems, or `NULL`)
- Whether a `current_flag`/`is_current` column is required for query
  convenience alongside the date range
- Whether changes are detected via full-row comparison or column-specific
  change detection (matters for ELT design — see Phase 4)

**Where the logic lives**: SCD handling (especially Type 2/3) is typically
implemented in the ELT layer (Snowflake Streams + Tasks, dbt snapshot,
MERGE statements against staging), not hand-maintained in the model
definition. Document both the target table structure (Phase 3) and the
load logic that populates it (Phase 4) — they are two different artifacts
that must stay consistent.

## Conformed dimensions

A dimension is **conformed** when it has the same structure, keys, and
meaning wherever it's reused across multiple fact tables/subject areas
(e.g. one `dim_date`, `dim_customer`, `dim_product` shared by Sales,
Risk, and Servicing fact tables). Conformed dimensions are what makes
cross-subject-area reporting and a "single version of truth" possible
instead of every mart reinventing its own Customer table.

When modeling a new subject area:
1. Check whether a dimension you're about to create already exists as a
   conformed dimension elsewhere in scope
2. If yes, reuse it — do not create a near-duplicate with slightly
   different attributes or grain
3. If a similar-but-not-identical dimension exists, **ask the user**
   whether it should be extended/conformed or is genuinely a distinct
   entity — don't silently merge or silently duplicate
4. Common conformed dimensions to look for by default: Date/Calendar,
   Customer, Product/Instrument, Geography/Branch, Account/Contract

## Hierarchical dimensions

If a dimension has a parent-child hierarchy (product category tree, org
structure, GL account hierarchy, geography), do not design it ad hoc —
read `07-hierarchical-dimensions.md` for the project's default pattern
(level-flattened, one row per node per level, with `is_complete_hierarchy_flag`
and SCD Type 2 versioning) and the alternatives (parent-child recursive,
bridge table, fixed-depth) with guidance on which to pick.

## Star vs. snowflake schema

| | Star schema | Snowflake schema |
|---|---|---|
| Dimension structure | Denormalized — each dimension is one flat table | Normalized — dimension broken into related sub-tables (e.g. `dim_product` → `dim_product_category`) |
| Query performance | Fewer joins, generally faster, simpler for BI tools | More joins, can be slower, more complex for end users |
| Storage | More redundancy | Less redundancy |
| Default recommendation | **Default choice** for Snowflake + BI-tool consumption (Power BI, Tableau, Streamlit) — intuitive for business users, plays well with Snowflake's columnar storage/compute model | Use selectively where a dimension has a large, independently-changing hierarchy (e.g. a deep, frequently-updated product taxonomy) and normalizing meaningfully reduces redundancy/update anomalies |

Default to star schema unless there's a specific, stated reason to
normalize part of a dimension. If proposing snowflaking any dimension,
state the reason explicitly rather than doing it by habit.

## Normalized (OLTP-style) vs. dimensional (OLAP-style) source data

Source systems are typically normalized (3NF) OLTP data optimized for
transactional writes, not analytical reads. Recognize this distinction
explicitly during Phase 0/1: normalized source tables are the *input* to
this process, not the target. The target of Phases 2–3 is always a
denormalized, query-optimized dimensional model, regardless of how
normalized the source is.
