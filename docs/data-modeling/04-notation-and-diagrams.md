# Notation & Diagramming

Use **crow's foot notation** at every stage (conceptual, logical, physical)
per the user's requirement — it's the most widely supported and readable
notation for relational/dimensional ERDs.

## Crow's foot symbol reference

| Symbol at the "many" end | Meaning |
|---|---|
| Crow's foot (three-pronged fork) | "many" |
| Single perpendicular tick | "one" |
| Circle (O) | "zero" (optional) |
| Double tick (\|\|) | "one and only one" (mandatory) |
| Circle + crow's foot | "zero or many" (optional many) |
| Tick + crow's foot | "one or many" (mandatory many) |

Reading a relationship line: read cardinality **at the far end from the
entity you're standing at**. E.g. `Customer ||--o{ Order` reads as "one
Customer has zero or many Orders" and "each Order belongs to exactly one
Customer."

Core elements at each stage:

| Element | Conceptual | Logical | Physical |
|---|---|---|---|
| Entities | Rectangle, name only | Rectangle, name + attribute list | Rectangle, table name + column list + data types |
| Attributes | Not shown | Listed inside entity, PK/FK marked | Listed with data type, nullability, PK/FK marked |
| Relationships | Line with cardinality | Line with cardinality + relationship verb | Line with FK constraint, matches DDL |
| Keys | Not shown | Logical PK/FK identified conceptually | Physical surrogate/natural keys per `03-naming-conventions.md` |
| Technology | None | None | Snowflake-specific types, schemas |

## How to actually render the diagrams in this environment

Two options, pick based on context:

1. **Inline in the chat response** — use the Visualizer tool
   (`visualize:show_widget`) to render an SVG ERD with proper crow's foot
   markers. Read the `diagram` module via `visualize:read_me` first. This
   is the right choice when the user wants to see and discuss the diagram
   conversationally.

2. **As a saved/shareable Mermaid ERD** — Mermaid's `erDiagram` syntax
   natively uses crow's-foot-equivalent cardinality markers and can be
   embedded in a markdown artifact/documentation file so it renders
   wherever the docs are viewed (GitHub, Confluence-with-mermaid, etc.):

   ```mermaid
   erDiagram
       DIM_CUSTOMER ||--o{ FACT_TRANSACTIONS : "places"
       DIM_PRODUCT ||--o{ FACT_TRANSACTIONS : "sold in"
       DIM_DATE ||--o{ FACT_TRANSACTIONS : "occurs on"

       DIM_CUSTOMER {
           number customer_sk PK
           string customer_id
           string customer_name
           string segment
           date valid_from
           date valid_to
           boolean is_current
       }
       FACT_TRANSACTIONS {
           number transaction_sk PK
           number customer_sk FK
           number product_sk FK
           number date_sk FK
           number transaction_amount
           number quantity
       }
   ```

   Mermaid cardinality tokens map to crow's foot as follows:
   `|o` zero-or-one, `||` exactly-one, `}o` zero-or-many, `}|` one-or-many.

   Use this for the **Physical Data Model documentation artifact** so it
   travels with the markdown doc. Use the Visualizer for the in-chat
   conceptual/logical walkthroughs during design discussion.

For the **conceptual model**, keep the Mermaid/SVG diagram to entities and
relationship lines only — do not add the `{ }` attribute block yet, to
stay true to "no attributes at conceptual stage."

## Other notations (context only — default to crow's foot)

Mention these only if the user asks or if source documentation already
uses one of them, in which case translate for consistency:

- **Chen's notation**: diamonds for relationships, ovals for attributes —
  more academic, rarely used in enterprise Snowflake projects
- **UML class diagrams**: from software engineering, models data as
  classes with attributes/operations — useful if the model needs to
  integrate with an object-oriented application layer
- **Barker's notation**: enterprise-oriented, combines ER concepts with
  simplified symbols — occasionally seen in Oracle-heritage shops
