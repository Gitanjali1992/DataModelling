# Hierarchical Dimensions (Ragged / Variable-Depth)

Many dimensions aren't flat — they have a parent-child hierarchy that can
be uneven ("ragged") in depth: a product tree might have some branches
five levels deep and others that stop at level 2. This file defines the
**level-flattened hierarchy pattern** to use by default in this project,
how it combines with SCD Type 2, how fact tables join to it, and the
alternative patterns to reach for when this one doesn't fit.

## The pattern: one row per node, cumulative ancestor columns

For a hierarchy with a maximum depth of N levels, every node in the tree
(at any depth) gets **its own row**. That row carries:

- `main_value` — the value of the node this row represents (equal to
  whichever `level_n` column is its deepest populated level)
- `level_1 … level_n` — the full ancestor path down to this node,
  populated from `level_1` through the node's own depth; levels deeper
  than this node's own depth are left null
- `is_complete_hierarchy_flag` — see semantics below
- `valid_from`, `valid_to`, `active_ind` — SCD Type 2 versioning applied
  at the node level (see below)

### Example (5-level hierarchy)

| main_value | level_1 | level_2 | level_3 | level_4 | level_5 | is_complete_hierarchy_flag | valid_from | valid_to | active_ind |
|---|---|---|---|---|---|---|---|---|---|
| L1 | L1 | | | | | N | 2024-01-01 | 9999-12-31 | Y |
| L2 | L1 | L2 | | | | N | 2024-01-01 | 9999-12-31 | Y |
| L3 | L1 | L2 | L3 | | | N | 2024-01-01 | 9999-12-31 | Y |
| L4 | L1 | L2 | L3 | L4 | | N | 2024-01-01 | 9999-12-31 | Y |
| L5 | L1 | L2 | L3 | L4 | L5 | Y | 2024-01-01 | 9999-12-31 | Y |

Every ancestor of a deep node also exists as its own standalone row, at
its own depth. This means the dimension can be filtered or joined at
*any* level without a recursive query — `WHERE level_2 = 'X'` returns
every row (at every depth) that descends from `X`, and a report that only
needs level-3 granularity can filter to rows where `level_3` is populated
and `level_4`/`level_5` are null.

### `is_complete_hierarchy_flag` — clarify the definition before building

This flag can mean one of two different things, and they lead to
different ETL logic. **Ask the user which definition applies** — don't
assume:

1. **"Reached the maximum defined depth"** — flag is `Y` only for rows at
   level N (the deepest level the hierarchy schema allows), regardless of
   whether that branch of the business hierarchy could theoretically go
   deeper. This is the simpler, structural interpretation, and matches
   the example above (only the level-5 row is flagged `Y`).
2. **"This is a true leaf in the source data"** — flag is `Y` for any row
   that has no children in the source hierarchy, even if that's at level
   2 or level 3 (a genuinely ragged hierarchy where not every branch goes
   to level 5). Under this definition, a level-2 node with no children
   would also get `is_complete_hierarchy_flag = 'Y'`.

These produce materially different results for ragged hierarchies, so
confirm with the business/data owner which one is intended, and document
the decision in the Logical Data Model doc.

## Combining with SCD Type 2

Since this pattern already tracks `valid_from` / `valid_to` / `active_ind`
per node, hierarchy changes over time are handled the standard SCD Type 2
way, but note the extra wrinkle: **a change to a node's position in the
hierarchy can cascade to every row that includes it as an ancestor.**

Example: if `level_2 = 'Electronics'` gets renamed or moved under a
different `level_1`, every existing row where `level_2 = 'Electronics'`
(its own row, plus every level_3/4/5 descendant row that carries
`level_2 = 'Electronics'` in its ancestor path) needs to be end-dated and
replaced with new rows reflecting the change. Decide and document with
the user:

- Does a rename/move at level 2 trigger new rows only for that node, or
  for every descendant row too? (Usually: every descendant row, since
  their `level_2` column value is now stale.)
- What's the change-detection method in the ELT layer — compare the full
  ancestor-path column set per `main_value`, not just the node's own
  attributes, since a change further up the tree changes descendant rows
  without the descendant's own "identity" changing

## Physical implementation (Snowflake)

```sql
CREATE TABLE dwh.dim_product_hierarchy (
    hierarchy_sk                 NUMBER      NOT NULL,   -- surrogate key
    main_value                    VARCHAR(200) NOT NULL,   -- this node's own value
    hierarchy_level_no             NUMBER       NOT NULL,   -- 1..N, this node's depth (recommended addition — confirm with user)
    level_1                         VARCHAR(200),
    level_2                         VARCHAR(200),
    level_3                         VARCHAR(200),
    level_4                         VARCHAR(200),
    level_5                         VARCHAR(200),
    is_complete_hierarchy_flag       VARCHAR(1)   NOT NULL,  -- Y/N — see definition above
    valid_from                       DATE         NOT NULL,
    valid_to                          DATE         NOT NULL,
    active_ind                        VARCHAR(1)   NOT NULL,  -- Y/N
    _source_system                    VARCHAR(50),
    _loaded_at                         TIMESTAMP_NTZ,
    CONSTRAINT pk_dim_product_hierarchy PRIMARY KEY (hierarchy_sk)
);
```

`hierarchy_level_no` (the numeric depth, 1 to N) isn't in the original
spec but is recommended: it lets queries filter to "give me only the
level-3 rows" with `WHERE hierarchy_level_no = 3` instead of the less
robust `WHERE level_3 IS NOT NULL AND level_4 IS NULL`. Propose it, but
confirm before adding — it changes the row shape slightly from what was
specified.

## How fact tables join to this dimension

Decide explicitly, per fact table, **which depth the fact's foreign key
points to**:

- If the fact is captured at leaf-level detail, `fact.hierarchy_sk` should
  point to the leaf-level row (`is_complete_hierarchy_flag = 'Y'`, per
  whichever definition was confirmed above)
- If a report needs to aggregate at an intermediate level (e.g. "sales by
  level_2 category"), that's normally handled by joining the fact's
  leaf-level FK to this dimension and then grouping by `dim.level_2` in
  the query/semantic layer — **not** by pointing the fact's FK at an
  intermediate-level row. Pointing fact FKs at multiple different depths
  of the same dimension is a common source of silently wrong aggregates
  (some fact rows resolve to level 3, others to level 5, and a naive
  `GROUP BY level_2` double-counts or under-counts). Flag this risk
  explicitly if the source data seems to require it.

## Alternative patterns (use these instead if they fit better)

This level-flattened, one-row-per-node pattern is one valid approach, but
it isn't the only one. Ask which fits before defaulting:

| Pattern | How it works | Best fit |
|---|---|---|
| **Level-flattened (this file's default)** | One row per node per level, cumulative ancestor columns, as above | Reporting needs equality filters at arbitrary levels without recursive SQL; hierarchy depth is bounded and known |
| **Parent-child (recursive) dimension** | Each row is one node with a `parent_sk` self-referencing FK; depth is unbounded and derived via recursive CTE (`WITH RECURSIVE` / Snowflake's `CONNECT BY`) | Hierarchy depth is unbounded or frequently changes shape; org charts, GL account trees |
| **Fixed-depth single-row-per-leaf** | One row per leaf node only, with `level_1…level_n` columns fully populated for that leaf's ancestor path — no separate rows for intermediate nodes | Hierarchy is not ragged (every branch reaches the same depth) and only leaf-level reporting is needed |
| **Kimball hierarchy bridge table** | Separate bridge table with one row per ancestor-descendant pair plus a "levels removed" column, used alongside a flat dimension | Complex ragged hierarchies with many-to-many reporting needs (e.g. an employee reporting to multiple committees) |

If the user's stated pattern (level-flattened) is being applied to a
hierarchy that turns out to be unbounded in depth or highly irregular,
flag that the pattern may not scale cleanly and confirm before proceeding
rather than forcing it.
