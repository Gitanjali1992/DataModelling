# Source Data Profiling Checklist

Run this checklist against every source table/file before any modeling
starts. Where you have Snowflake query access, run the SQL directly.
Where you don't, ask the user to run it and paste back results, or ask for
a data dictionary / sample export. Never model on assumption.

## 1. Structural inventory

For each table, capture:

- Column list, declared data type, nullability, default
- Observed vs. declared type (a `VARCHAR` column holding only digits is a
  smell — flag it, don't silently treat it as numeric)
- Format consistency within a column: date formats, casing (e.g.
  `P123` vs `p123`), unit consistency (cents vs. dollars, `%` vs decimal),
  code/enum consistency (e.g. `Y/N` vs `1/0` vs `true/false` mixed)
- Candidate keys: columns that look unique — verify, don't trust the name

```sql
-- Column-level type & null overview
SELECT column_name, data_type, is_nullable, character_maximum_length
FROM information_schema.columns
WHERE table_schema = '<SCHEMA>' AND table_name = '<TABLE>'
ORDER BY ordinal_position;
```

## 2. Null / completeness analysis

Run per column, not just per table — a table can be "90% complete"
overall while one critical column is 60% null.

```sql
SELECT
  COUNT(*)                                            AS row_count,
  COUNT(*) - COUNT(customer_id)                        AS null_customer_id,
  COUNT(*) - COUNT(email)                               AS null_email,
  ROUND(100 * (COUNT(*) - COUNT(email)) / COUNT(*), 2) AS pct_null_email
FROM <schema>.<table>;
```

For a fast full-table sweep, generate one query per column dynamically
(or use `SELECT * EXCLUDE (...)` patterns / a profiling utility) rather
than hand-writing dozens of `COUNT(col)` lines.

For every column with a non-trivial null rate, determine and record:
- **Structural null**: field legitimately doesn't apply for that row
  (e.g. `middle_name`, `promo_code` on non-promo orders) — not a defect
- **Defect null**: field should be populated but isn't (source system
  bug, failed integration, late-arriving data) — flag for remediation and
  decide default/exclusion handling with the user
- If unsure which category a null falls into, **ask** — don't guess

## 3. Referential integrity analysis

Check every FK relationship implied by the model, even if Snowflake
doesn't enforce it physically.

```sql
-- Orphaned foreign keys: rows in the "many" side with no matching parent
SELECT f.order_id, f.customer_id
FROM fact_orders f
LEFT JOIN dim_customer d ON f.customer_id = d.customer_id
WHERE d.customer_id IS NULL;

-- Duplicate values in a column expected to be a unique/primary key
SELECT customer_id, COUNT(*) AS cnt
FROM dim_customer
GROUP BY customer_id
HAVING COUNT(*) > 1;

-- Cardinality check: confirm an assumed 1:N relationship isn't secretly M:N
SELECT order_id, COUNT(DISTINCT customer_id) AS distinct_parents
FROM fact_orders
GROUP BY order_id
HAVING COUNT(DISTINCT customer_id) > 1;
```

Record: which relationships are clean, which have orphans (and the %),
which "unique" keys aren't actually unique, and any cardinality that
contradicts what the source documentation/business claimed. Broken
referential integrity discovered here directly affects Phase 2 grain and
key decisions — do not defer fixing it to "later."

## 4. PII / confidential data screening

Classify **every column** against this taxonomy before it's allowed into
any dimension or fact design. Do not rely on column naming alone — sample
values, since PII is sometimes hidden in generically-named free-text or
comment fields.

| Class | Examples | Default handling (confirm with user) |
|---|---|---|
| Direct identifier | Name, SSN/PAN/Aadhaar/national ID, passport no., email, phone, physical address, DOB | Mask/tokenize in non-prod, restrict access via RBAC + row/column-level security in prod |
| Financial/BFSI-sensitive | Account number, IBAN/SWIFT, card PAN, balance, income, credit score, holdings, transaction amounts tied to an identified person | Dynamic Data Masking + strict RBAC; consider whether it belongs in a conformed dimension at all vs. a restricted-access mart |
| Quasi-identifier | ZIP/postal code, DOB (without name), gender, employer, IP address | Consider generalization/binning (e.g. age band instead of DOB) if used broadly |
| Special-category / sensitive | Health data, religion, ethnicity, biometric, criminal record, union membership | Treat as highest sensitivity; confirm legal basis and access model with user/compliance before modeling |
| Confidential-but-not-personal | Pricing, contracts, internal strategy fields, unreleased financials | Restrict via RBAC/schema grants; not a PII masking problem but still an access-control one |
| Non-sensitive | Product category, transaction date, quantity, order status | Model normally |

For every flagged column, produce: column name, source table, PII class,
sample-based confidence (did you infer this from name, values, or both),
and a recommendation. **Then ask the user how they want each class
handled** (mask at ingestion, tokenize, exclude from the model entirely,
restrict via Snowflake object/row/column-level policies) — this is a
governance decision, not a modeling default you set unilaterally.

Relevant Snowflake mechanisms to mention when discussing handling options:
Dynamic Data Masking policies, Row Access Policies, tag-based masking,
object-level RBAC, and (for regulated BFSI data) column-level
classification tags for audit/compliance reporting.

## 5. KPI opportunity analysis

Goal: don't just model the data as given — identify what business value
could be extracted from it, then let the user/business confirm priority.

Method:
1. **Inventory measures present or derivable**: numeric/quantitative
   columns in the source, plus anything derivable via simple arithmetic
   (e.g. `quantity * unit_price`), ratios, or time-based comparisons
2. **Inventory grain and dimensions available**: what can this be sliced
   by (time, product, customer, region, channel, account type, etc.)?
3. **Pattern-match against common KPI families** for the domain — for
   BFSI specifically, consider (as candidates, not conclusions):
   - Volume/growth: transaction volume, AUM/AUA growth, new account
     openings, portfolio inflows/outflows
   - Profitability: net interest margin, fee income, cost-to-income ratio
   - Risk/quality: delinquency rate, NPA ratio, exception/error rate
   - Customer: retention/attrition, product penetration, NPS-linked
     metrics if survey data exists, time-to-onboard
   - Operational: SLA adherence, straight-through-processing rate,
     reconciliation break rate
4. **For each candidate KPI**, document: proposed definition, required
   measures, required grain, required dimensions, additivity type (see
   `02-dimensional-modeling-guide.md`), and any data quality gaps found in
   steps 2–4 above that would block it (e.g. "blocked: 40% null on
   `close_date`")
5. Present the candidate list to the user/business as a **Candidate KPI
   Register** for prioritization — do not unilaterally decide which KPIs
   the model should support. Their answer feeds directly into which facts
   and dimensions get built first.

Output of this phase feeds the Data Profiling & Risk Report template in
`05-documentation-templates.md`.
