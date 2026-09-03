# Clarification Protocol

The point of this skill is to prevent an agent from confidently producing
a plausible-looking but wrong data model. Use this list as a trigger set —
if any of these situations arise, stop and ask before proceeding, rather
than choosing the "reasonable default" and pressing on.

## When to always ask

1. **Grain is ambiguous or the data mixes grains.** If a fact source could
   plausibly be "one row per order" or "one row per order line," ask.
   Don't infer from row count alone.
2. **A measure's additivity isn't obvious from name/values alone.** Ratios
   and percentages are usually clear, but things like "average balance,"
   "count," or business-specific ratios may not be — confirm the intended
   aggregation behaviour with the business.
3. **An attribute could plausibly need SCD Type 1 or Type 2.** If change
   history matters for some reports but not others, ask which reports
   drive the decision, rather than picking Type 2 "to be safe" (which has
   real storage/complexity cost) or Type 1 "to be simple" (which silently
   destroys history).
4. **Two entities/columns might represent the same real-world thing** but
   come from different source systems with different names, formats, or
   granularity (e.g. "Client" in CRM vs. "Customer" in core banking). Ask
   whether they should conform into one dimension.
5. **A column might be PII/confidential.** If in doubt whether a field
   (especially free-text, comments, or generically-named columns) contains
   personal or confidential data, flag it and ask rather than assuming
   it's safe because the column name looks generic.
6. **Null values could be structural or a defect**, and the distinction
   changes downstream handling (e.g. default value vs. exclusion vs.
   remediation request to the source team).
7. **Referential integrity is broken** and there's more than one plausible
   fix (drop orphans, create an "unknown" placeholder dimension row,
   escalate to source system owner). Ask which approach the business
   wants — this affects reported totals.
8. **A candidate KPI could be defined more than one way** (e.g. "active
   customer" — active in last 30/60/90 days? by transaction or by login?).
   Never silently pick a definition for a business term.
9. **Naming conventions conflict** between this skill's default (see
   `03-naming-conventions.md`) and an existing house standard the user
   mentions or that's visible in existing objects. Ask which wins.
10. **Star vs. snowflake for a specific dimension** isn't a clear default
    case (e.g. a deep, independently-maintained hierarchy). State your
    recommendation and reasoning, then confirm rather than just building it.
11. **Historical load scope is unclear** — how far back should historical
    load go, and is there a defined cutover point between historical and
    incremental load.
12. **A hierarchical dimension's `is_complete_hierarchy_flag` meaning is
    ambiguous** — "reached max defined depth" vs. "true leaf in the
    source data" produce different flag values for ragged hierarchies.
    Confirm before loading (see `07-hierarchical-dimensions.md`). Also
    confirm whether a mid-hierarchy change should cascade new rows to all
    descendant nodes.
13. **Deletion handling in source is ambiguous** — logical vs. physical
    deletes, and whether deleted records should ever be purged from the
    warehouse. Confirm the business rule per source system rather than
    applying a blanket policy.

## How to ask

- Batch related questions together in one turn rather than a slow back
  and forth — use `ask_user_input_v0` (tappable options) if the tool is
  available and the question has a small number of discrete choices; use
  plain text questions for open-ended ones (e.g. exact KPI definitions).
- State your best-guess recommendation alongside the question when you
  have one, so the user can quickly confirm rather than having to
  originate the answer from scratch: "I'd default to SCD Type 2 for
  `customer.address` since address history matters for regional reporting
  — confirm, or would Type 1 be sufficient here?"
- Keep momentum: ask what's genuinely blocking, keep profiling/documenting
  everything that isn't blocked, and come back to open questions rather
  than halting all progress on one unresolved point.
- Record every question and its answer in the relevant documentation
  artifact's "Open Questions" / "Assumptions Confirmed" section (see
  `05-documentation-templates.md`) so the decision trail is preserved.

## What NOT to do

- Do not silently pick SCD Type 1 because it's simpler to implement
- Do not silently assume a column is not PII because its name looks
  generic or because checking is inconvenient
- Do not silently merge two similarly-named entities, or silently treat
  them as unrelated, without asking
- Do not silently drop orphaned foreign key rows without flagging the
  referential integrity issue and getting a decision on handling
- Do not present a candidate KPI list as final — it is always a proposal
  for business prioritization
