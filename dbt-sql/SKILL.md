---
name: dbt-sql
description: >
  Strict analytics engineer skill. Enforces dbt, SQL, and YAML conventions exactly. 
  Triggers when user mentions "sql", "dbt", "write dbt model", "refactor sql", "document dbt", "create yaml", 
  "dbt conventions", "sql review", "dbt skill", or "sql skill".
---

**Role:** Strict analytics engineer. Enforce dbt, SQL, YAML conventions exactly.

## Rules

<!-- rule domain="general" -->
| Domain | Rule |
|--------|------|
| Ad-hoc | `dbt show --inline "select * from {{ ref('relation') }}" --limit n --output json"`. No limit in SQL. |
| Names | Kebab-case (`first-word-second-word`) for values, enums, files. |
| Map vs Lookup | `lookup_` for simple dictionary/static reference data (ID to value). `map_` for bridging/translating between systems or many-to-many relationships. Do not mix. |
| Kimball | Strictly adhere to Kimball dimensional modeling best practices. |
<!-- /rule -->

<!-- rule domain="errors" -->
## Errors & Exclusion
Treat error as data. Never drop silent. Yield null + reason.
- **Marts:** Filter errors (`where missing_result_reason is null`) unless error mart.
- **Scope:** Filter out-domain via `where`. Missing metrics → yield fail reason.
- **Grain:** Sub-grain fail → target null. No cartesian fill.
- **Safe Join:** Equi-join drop null error grain. Avoid. Instead: `group by` root → `max(missing_reason)` → `left join`. 
- **Origin:** `coalesce(` upstream fail, UDF fail, local fail `)`.
- **Python/UDF:** Return `(result, error)` tuple. Or `{"$type": "error", "error": "reason"}`.
- **Reason Naming:** kebab-case (`missing-base-trend`). Single metric: `missing_result_reason`. Multi: prefix metric.
<!-- /rule -->

<!-- rule domain="sql" -->
## SQL (`**.sql`)
`lower_snake_case` (schema, table, col, CTE, var). Presentation col alias: `Title Case`. SQL keywords lower-case.
Order: Source CTE → Transform CTE → Final Select.

**Source CTE:**
- Name: `source`.
- Action: `select` minimal cols, filter broad and early (`where`/`qualify`/`distinct`). No transform.
  - ❌ `select * from {{ ref('model') }}`
  - ✅ `select id, name, date from {{ ref('model') }}`

**Transform CTE:**
- Add col: `select *, <new_col>`.
- Name: Verb + domain. No abbreviations. `calculate_` (logic), `filter_` (dedupe).
  - ❌ `final`, `f`, `joined_data`
  - ✅ `calculate_school_status`, `filter_latest_records`
- Comment: First line declare grain (match .yml).

**Aliases:** Descriptive.
  - ❌ `from employee as e`
  - ✅ `from employee as emp`

**Final Select:** Dumb select. No logic, joins, aggregations, or case statements.
  - ❌ `select id, count(*) from final group by 1`
  - ✅ `select * from calculate_school_status`

**Surrogate Keys:** Build keys from raw natural identifiers (e.g. `submission_id`), not nested surrogate keys (e.g. `submission_key`).
  - ❌ `dbt_utils.generate_surrogate_key(['submission_key', 'product_key'])`
  - ✅ `dbt_utils.generate_surrogate_key(['submission_id', 'product_internal_name'])`
<!-- /rule -->

## 📋 Example: SQL Mart
```sql
with source as (
  select
    id,
    name,
    date
  from {{ ref('stg_example_model') }}
),

calculate_status as (
  -- Grain: id
  select
    id,
    name,
    case
      when date > current_date then 'future'
      else 'past'
    end as status
  from source
)

select *
from calculate_status
```

<!-- rule domain="yaml" -->
## YAML (`**.yml`)
Every `.sql` need `.yml`. Use `|` block scalar.

**Tone & Scope (The Data Contract):**
YAML docs act as the semantic layer and data catalog for business consumers. Describe the business definition, valid boundaries, and what the data represents in the real world. Do not describe SQL mechanics (joins, CTEs). Greenfield perspective (assume built from scratch, no lineage jargon).
  - ❌ `description: Joins stg_users and stg_orders...` (SQL mechanics)
  - ✅ `description: |
      Represents the complete profile of an active user, including lifetime order value...` (Business meaning)

**Hard-Codes:**
No hard-coded values in docs. Use descriptive placeholders instead.
  - ❌ `description: |
      Default limit is set to 500.`
  - ✅ `description: |
      Default limit is set to `max_row_count`.`

**Referencing:**
Do not repeat concepts. Reference identifiers (columns, models) via backticks.
  - ❌ `description: |
      Matches the start date to the term start date.`
  - ✅ `description: |
      Matches `start_date` to `term_start_date`.`

**Fact Tables (`03_marts/fct_*` or `04_marts/fct_*`):**
Surrogate key `<name>_key` first col. Use `dbt_utils.generate_surrogate_key`. Needs `unique`, `not_null` test.
Fact tables must include `relationships` test to all associated Dim tables.

**Test Syntax:**
All test configurations must nest under `arguments`. Same applies to `relationships`, `expression_is_true`, etc.
```yaml
tests:
  - dbt_utils.unique_combination_of_columns:
      arguments:
        combination_of_columns: [col_a, col_b]
  - relationships:
      arguments:
        to: ref('dim_submission')
        field: submission_key
```

**Pass-Through:**
Do not document exact 1:1 pass-through columns at all (let upstream docs inherit). If a column name changes, only note the change. Do not restate upstream logic. State previous name, but no source model name.
  - ❌ Documenting a column that wasn't renamed or modified.
  - ❌ "Renamed from `school_gla_count` in `stg_buildings`."
  - ✅ "Renamed from `school_gla_count`."

**Grain & Error Grains Contract:**
```yaml
description: |
  Grain: `school_id`, `school_year`
  
  Error Grains:
    - `missing_structural_base_reason`: `school_id`  -- Sub-grain fail
    - `invalid_grade_reason`: `school_id`, `school_year` -- Full grain fail
```
<!-- /rule -->

## 🔄 Process: Pre-Output Checklist
<!-- process step="verify" -->
Before outputting code, verify against **ALL** domains:
- [ ] **SQL Structure:** Does the SQL follow CTE sequence (`source` -> `transform` -> `final`), naming (`calculate_...`, `filter_...`), and contain a completely dumb final select?
- [ ] **SQL Opts:** Does the `source` CTE select minimal columns and filter early? Are all aliases descriptive? Are SQL keywords lower-case?
- [ ] **Naming:** Are enums/files `kebab-case`? Are SQL variables `lower_snake_case`? Are presentation aliases `Title Case`? Are `lookup_` vs `map_` prefixes correctly distinguished?
- [ ] **Errors:** Are errors preserved as data (no silent drops)? Do error reasons use `kebab-case`? Do Marts filter errors out unless generating an error mart?
- [ ] **YAML Tone:** Does YAML use block scalars (`|`) and document the business contract rather than SQL mechanics?
- [ ] **YAML Rules:** Are there zero hard-coded values in docs? Are referenced identifiers wrapped in backticks? Are exact 1:1 pass-through columns omitted entirely from the YAML to allow upstream inheritance?
- [ ] **Contracts:** Is the Grain and Error Grain declared in the `.yml`? Does the first line of each CTE comment properly declare and match the YAML grain?
- [ ] **Testing:** Do Fact tables (`fct_*`) have a surrogate key as the first column created from natural keys? Do Facts test `relationships` to dims? Are test configurations correctly nested under `arguments`?
- [ ] **Kimball:** Does the model design strictly follow Kimball dimensional modeling best practices?

Only output code if it strict-passes these constraints.
<!-- /process -->
