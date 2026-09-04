# Documentation Templates

Use these templates verbatim (adapt headers as needed) so documentation is
consistent across datasets and projects. Produce each as a markdown
artifact so it's saved and shareable, not just conversational output.

Save each completed template to `docs/<subject-area>/` using the file
names in the phase table in `SKILL.md` (e.g.
`docs/customer-transactions/00-profiling-report.md`) — see "Output
location convention" there for the full rule.

---

## Template A — Data Profiling & Risk Report (Phase 0 output)

```markdown
# Data Profiling & Risk Report — <source system / dataset name>
Date: <date> | Analyst: <name/agent>

## 1. Scope
Tables/files profiled: <list>

## 2. Structural Findings
| Table | Column | Declared Type | Observed Issue | Recommendation |
|---|---|---|---|---|

## 3. Null / Completeness Analysis
| Table | Column | Null % | Structural or Defect? | Recommendation |
|---|---|---|---|---|

## 4. Referential Integrity Findings
| Relationship | Orphan Rows Found | % of Total | Root Cause (if known) | Recommendation |
|---|---|---|---|---|

## 5. PII / Confidential Data Classification
| Table | Column | PII Class | Confidence (name/value/both) | Recommended Handling | Confirmed by Business? |
|---|---|---|---|---|---|

## 6. Candidate KPI Register
| KPI Name | Definition | Required Measures | Required Grain | Required Dimensions | Additivity | Blocking Data Issues | Business Priority (TBC) |
|---|---|---|---|---|---|---|---|

## 7. Open Questions for Business/Data Owner
- <question 1>
- <question 2>
```

---

## Template B — Conceptual Data Model Doc (Phase 1 output)

```markdown
# Conceptual Data Model — <subject area>

## Entities
| Entity | Business Definition |
|---|---|

## Relationships
| From Entity | Relationship | To Entity | Cardinality |
|---|---|---|---|

## Diagram
<crow's foot ERD, entities + relationships only, no attributes>

## Assumptions Confirmed With Business
- <assumption> — confirmed by <who>, on <date>
```

---

## Template C — Logical Data Model Doc (Phase 2 output)

```markdown
# Logical Data Model — <subject area>

## Fact Table: <fact_name>
- **Grain**: <one sentence — confirmed with business on <date>>
- **Measures**:
  | Measure | Definition | Additivity | Notes |
  |---|---|---|---|
- **Foreign Keys**: <list of dimensions this fact connects to>

## Dimension Table: <dim_name>
- **Business Definition**: <what this dimension represents>
- **Natural Key**: <source system business key>
- **Conformed?**: Yes/No — shared with: <other fact tables/subject areas>
- **Attributes & SCD Typing**:
  | Attribute | Business Definition | SCD Type | Rationale |
  |---|---|---|---|

(Repeat Fact/Dimension blocks for every object in scope)

## Cardinality Summary
| Entity A | Relationship | Entity B | Cardinality | Notes |
|---|---|---|---|---|

## Star vs. Snowflake Decision
<which schema style was used per dimension, and why>

## Hierarchical Dimensions (if applicable)
For each hierarchical dimension, document (per `07-hierarchical-dimensions.md`):
- Pattern used: Level-flattened / Parent-child recursive / Bridge table / Fixed-depth — and why
- Maximum depth (N levels) and whether the hierarchy is ragged
- `is_complete_hierarchy_flag` definition confirmed: max-depth-reached vs. true-leaf-in-data
- SCD Type 2 cascade rule confirmed: does a mid-hierarchy change regenerate descendant rows?
- Which depth fact tables' foreign keys join to (leaf-level vs. intermediate)

## .dbml File
<embed or link the .dbml file for this logical model — generic/logical
types, `Note:` fields carrying grain and SCD summaries, per the DBML
section in `04-notation-and-diagrams.md`>
```

---

## Template D — Physical Data Model Doc (Phase 3 output)

```markdown
# Physical Data Model — <subject area>
Target platform: Snowflake | Schema layer: <raw/stg/dwh/mart>

## DDL
<full CREATE TABLE statements, per 03-naming-conventions.md>

## ERD (Mermaid, crow's foot cardinality)
<mermaid erDiagram block per 04-notation-and-diagrams.md>

## Object Placement
| Object | Schema | Layer | Notes |
|---|---|---|---|

## Performance Considerations
| Object | Clustering Key | Rationale |
|---|---|---|

## PII/Access Control Objects
| Object | Type (masking policy / row access policy / tag) | Applied To | Rule Summary |
|---|---|---|---|
```

---

## Template E — Transformation & KPI Specification (Phase 4 output)

One entry per transformation or KPI. Always both business and technical
versions.

```markdown
### <KPI or Transformation Name>

**Business definition** (for business/analyst audience):
<plain-language explanation of what this measures and why it matters —
no SQL, no internal jargon>

**Technical definition** (for engineering audience):
- Source table(s)/column(s): <...>
- Join logic: <...>
- Filter logic: <...>
- Aggregation / calculation: <exact formula>
- Additivity type: Additive / Semi-additive / Non-additive
- SCD handling implications (if joined to a Type-2 dimension, specify
  point-in-time join logic): <...>
- Grain of output: <...>
- Sample SQL:
  ```sql
  <SQL>
  ```

**PII/masking notes** (if applicable): <...>

**Load type**: Initial / Historical / Incremental (delta vs. full extract)
— per source; note timestamp reliability if using delta detection

**Data quality caveats**: <any known gaps from the Phase 0 report that
affect this KPI's reliability, and how they're handled (excluded,
flagged, approximated)>
```

---

## General documentation rules

- Every artifact is a standalone markdown file — link between them rather
  than duplicating content
- Every artifact records **who confirmed what, and when** for any business
  decision (grain, SCD type, PII handling, KPI definition) — this is the
  audit trail that prevents "why did we model it this way" archaeology
  later
- If a decision is still open/unconfirmed, say so explicitly in the doc
  (`STATUS: pending business confirmation`) rather than presenting a
  guess as settled
