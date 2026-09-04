# Data Modeling & Source Analysis — Copilot Instructions

This repository builds dimensional (star schema) data models on Snowflake
from source data, following an ELT pattern. Apply these rules whenever
helping with data profiling, dimensional modeling, DDL, or documentation
in this repo.

## Golden rule: ask, don't assume

Data modeling decisions (grain, additivity, SCD type, whether a column is
PII, whether two entities should be one dimension) are business decisions,
not technical defaults. **When any of these is unclear from the data or
the request, ask a clarifying question before generating code or a
model.** Never silently pick a default for: fact table grain, measure
additivity, SCD type per attribute, PII classification, or whether a
hierarchy node counts as "complete."

## Workflow — five phases, in order

1. **Profile the source** — nulls/completeness, format consistency,
   referential integrity, PII/confidential screening, candidate KPI list.
   Save to `docs/<subject-area>/00-profiling-report.md`
2. **Conceptual model** — entities and relationships only, no attributes,
   confirm with the user before moving on. Save to
   `docs/<subject-area>/01-conceptual-model.md`
3. **Logical model** — facts (grain, measures, additivity) and dimensions
   (attributes, SCD type per attribute, conformed or not). Also produce a
   `.dbml` file for this stage (see the notation doc below). Save to
   `docs/<subject-area>/02-logical-model.md` (+
   `02-logical-model.dbml` alongside it)
4. **Physical model (Snowflake)** — DDL following this repo's naming
   conventions, crow's foot ERD. Save to
   `docs/<subject-area>/03-physical-model.md`
5. **Transformation & KPI documentation** — every KPI and transformation
   documented twice: plain business language, then exact technical logic.
   Save to `docs/<subject-area>/04-transformation-kpi-specs.md`

## Output location convention

When a phase completes, **write its output as an actual markdown file**
at the path given above, don't just answer in chat and leave saving it
to the user. Replace `<subject-area>` with a short slug for whatever's
being modeled (e.g. `docs/customer-transactions/`). This is a *different*
`docs/` location from the reference library below — `docs/data-modeling/`
holds this project's fixed rulebook, while `docs/<subject-area>/` holds
the per-dataset outputs this rulebook produces. If `docs/<subject-area>/`
doesn't exist yet, create it. If a file at that path already exists
(re-running a phase after new information), update it rather than
creating a duplicate with a different name.

## Naming conventions (quick reference)

- `snake_case`, lowercase, no reserved words as identifiers
- Fact tables: `fact_<subject>`. Dimension tables: `dim_<entity>`
- Dimension surrogate key: `<entity>_sk`. Natural/business key: `<entity>_id`
- Fact table foreign keys match the referenced dimension's surrogate key
  name exactly (e.g. `customer_sk`)
- SCD Type 2 tracking columns: `valid_from`, `valid_to`, `is_current`
- ELT/audit metadata columns are prefixed with underscore: `_loaded_at`,
  `_source_system`

## Reference docs — full detail lives in `docs/data-modeling/`

These are **not** auto-loaded. Attach the relevant one to the chat (drag
it in, or type `#file:docs/data-modeling/<name>.md`) when you're working
on that part of the task:

| File | Use when |
|---|---|
| `01-source-profiling-checklist.md` | Profiling a new source: nulls, format issues, referential integrity, PII screening, KPI discovery |
| `02-dimensional-modeling-guide.md` | Designing facts/dimensions: grain, additivity, SCD types, conformed dimensions, star vs snowflake |
| `03-naming-conventions.md` | Full naming standard for schemas, tables, columns, keys, masking objects |
| `04-notation-and-diagrams.md` | Drawing conceptual/logical/physical ERDs in crow's foot notation (Mermaid), plus the `.dbml` file format for the logical model |
| `05-documentation-templates.md` | Writing up the profiling report, model docs, or KPI/transformation specs |
| `06-clarification-protocol.md` | Full list of situations that require asking before proceeding |
| `07-hierarchical-dimensions.md` | Modeling a parent-child hierarchy (product tree, org chart, GL accounts) |
| `08-semi-structured-source-formats.md` | Source is CSV, JSON, XML, or an API response rather than a clean table |

If unsure which doc is relevant, ask me which phase we're in and I'll
attach it — don't guess at profiling/modeling rules from general
knowledge when a doc in this folder covers it.
