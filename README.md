
Phase 0a — Profile the source
Attach 01-source-profiling-checklist.md. If your sample is CSV, JSON, XML, or an API dump rather than a clean table, also attach 08-semi-structured-source-formats.md. Type: "Using the attached source profiling checklist, profile [filename/table] for structural issues, null/completeness, referential integrity, and PII. Also list candidate KPIs. Produce a Data Profiling & Risk Report and ask me anything that's unclear."
2
Phase 1 — Conceptual model
Attach 04-notation-and-diagrams.md. Type: "Based on the profiling report, propose a conceptual data model in crow's foot notation — entities and relationships only, no attributes yet. Confirm the entities with me before we go further."
3
Phase 2 — Logical model
Attach 02-dimensional-modeling-guide.md. If the dataset has a parent-child hierarchy (product tree, org chart, etc.), also attach 07-hierarchical-dimensions.md. Type: "Using the attached dimensional modeling guide, design the logical model — confirm fact table grain, classify each measure's additivity, and assign an SCD type per dimension attribute. Ask me anything ambiguous rather than assuming."
4
Phase 3 — Physical model (Snowflake DDL)
Attach 03-naming-conventions.md (keep 04-notation-and-diagrams.md attached too). Type: "Using the attached naming conventions and notation guide, generate the Snowflake physical model — full CREATE TABLE DDL for the facts and dimensions, plus a crow's foot ERD in Mermaid."
5
Phase 4 — Transformation & KPI documentation
Attach 05-documentation-templates.md. Type: "Using the attached documentation template, write the Transformation & KPI Specification for [KPI name] — include both the business-language definition and the technical definition with SQL."
6
Optional — the standing rule, any time
06-clarification-protocol.md isn't tied to one phase, it's the full list of situations where Copilot should stop and ask instead of guessing. copilot-instructions.md already encodes the short version automatically, so you only need to attach 06 directly if you want to point Copilot at a specific trigger it seems to have missed, e.g.: "Check this against the clarification protocol attached — should you have asked me something here before proceeding?"
