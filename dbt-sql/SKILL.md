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
| Names | `lower_snake_case` (`first_word_second_word`) for values, enums, files. |
| Keys | Surrogate & foreign keys named `<entity>_key` (`risk_key`, `domain_key`). A dimension PK and every fact FK that references it share the *identical* `<entity>_key` name. Keep the natural/business id as `<entity>_id` (degenerate). |
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
- **Reason Naming:** lower_snake_case (`missing_base_trend`). Single metric: `missing_result_reason`. Multi: prefix metric.
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

**FK Resolution — prefer compute, join is the fallback:** Compute a fact/dim foreign key, don't join, whenever the key is derivable. Priority:
  1. **Single natural key** — use a narrow business value directly (e.g. `lower(trim(domain))`). No join, no hash needed.
  2. **Hash** — for composite or derived keys, always hash when the engine provides one (`dbt_utils.generate_surrogate_key`). The hash is narrow and deterministic; dim and fact compute the same key independently. Concatenate raw strings (`org || '|' || discipline`) only when no native hash exists — a long composite costs storage and comparison time.
  3. **Join** — only when the key is not computable: sequence/identity surrogates, survivorship/fuzzy-matched keys, or SCD2 point-in-time lookups.

Always provide an Unknown member (key `0` / `-1` or `'(unspecified)'`). Computed keys return the sentinel; left joins coalesce to it.
  - ❌ `left join dim_domain using (domain)` to fetch a key derivable as `lower(trim(domain))`.
  - ✅ `lower(trim(domain)) as domain_key` (natural key, no join).
  - ✅ `dbt_utils.generate_surrogate_key(['organisation', 'discipline']) as rbs_key` (hash, no join).
<!-- /rule -->

<!-- rule domain="sql" -->

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

**Sentence Form:**
Every doc body (inline `description`, `{% docs %}` block, hoisted clause body) is one or more complete sentences — capitalised, full-stopped, grammatical alone. Composition happens by concatenation: sentence + space + sentence, or sentence introduced by a colon. Never inline a doc body mid-sentence; capitalisation and full stops must survive.
  - ❌ `description: "{{ doc('prismg2__phase') }} for matching `*_description`."` → "Project lifecycle phase... for matching..." (mid-sentence period, dangling fragment).
  - ✅ `description: "{{ doc('prismg2__phase') }} {{ doc('prismg2__description') }}"` → two sentences, juxtaposed.

**Hoisting:**
Repeating col doc pattern within one model → hoist to model `description`. State rule once at model level. Per-col `description` empty, or only note deviation.
Repeating col doc pattern across 2+ models → doc block (see below), not hoist.

**Hoist sentence shape:**
- Paired (anchor exists): `` `<concept>_<form>` columns, anchored on `<concept>_<other_form>`: {{ doc('<scope>__<form>') }} ``
- Standalone (no anchor): `` `*_<form>` columns: {{ doc('<scope>__<form>') }} ``

A hoist line has two parts:
  1. **Identifier** — selector names which cols the rule covers. Use `<concept>_<form>` when a co-referring slot exists (the repeated `<concept>` token tells the reader the same prefix binds both sides, like a regex capture group). Use `*_<form>` when nothing co-refers.
  2. **Anchor qualifier (optional)** — `anchored on <concept>_<other_form>` names the sibling col that carries the per-instance concept. Present only when the form has such a sibling.
    - ✅ (paired) `` `<concept>_code` columns, anchored on `<concept>_description`: {{ doc('prismg2__code') }} `` → reader infers `phase_code` ↔ `phase_description`, `portfolio_code` ↔ `portfolio_description`, …
    - ✅ (standalone) `` `*_sort_order_ambiguous` flags: {{ doc('prismg2__sort_order_ambiguous') }} ``
    - ❌ (anchor dropped where a sibling exists) `` `*_code` columns: … `` → reader cannot reach the concept anchor.
    - ❌ (anchor forced where no sibling exists) `` `*_sort_order_ambiguous` flags, anchored on `*_description`: … `` → invents an anchor the model does not encode.

The `doc()` body must NOT repeat the form suffix — the selector (`*_<form>`) already names the form. Body describes what the form means, not which suffix carries it.
  - ❌ body: "Columns suffixed `_pbi_escaped` apply escape characters..." (suffix repeated)
  - ✅ body: "Escape characters applied to the raw description..." (form named once, by selector)

**Doc Blocks (`{% docs %}`):**
Use doc blocks for concepts referenced across 2+ models. Name with scope prefix:
  - `<source>__` — owned by one source system (e.g. `prismg2__workspace_id`).
  - `<domain>__` — owned by a business area, may span sources (e.g. `major_projects__excom_cost`).
  - `conformed__` — enterprise-wide conformed dim/measure (e.g. `conformed__date`).
  - `pattern__` — subject-agnostic modelling mechanic (e.g. `pattern__grain`, `pattern__pbi_escaped`).

File layout:
  - Source: `models/00_source/<source>.md` next to `<source>.yml`.
  - All other scopes: `models/_docs/<scope>.md` (one file per domain; `_conformed.md`, `_patterns.md` for project-wide).

Promotion: start narrow. Promote `<domain>__` → `pattern__` only when a second consumer exists (rule of two). Patterns live under the narrowest scope that owns them — a pattern used by one source stays in `<source>__` until a second source adopts it.

**Concept × Form Composition:**
Cols are often `<concept>_<form>` (e.g. `phase_code`, `phase_description`, `phase_description_pbi_escaped`). Concept and form are **separate doc blocks** — never fuse into one. Two cases:

1. **Concept appears bare in the model** (e.g. `workspace_id` is itself a col) → concept doc on the bare col. Any `<concept>_<form>` variants follow Hoisting (form mechanic hoists to model desc; per-col empty).
2. **Concept only suffixed** → pick **one** form col as the concept anchor (default `_description`; else most identity-bearing present: `_code` > `_name` > `_id`). Wire `description: "{{ doc('<scope>__<concept>') }} {{ doc('<scope>__<form>') }}"` there. All other forms for this concept follow Hoisting.

Form doc blocks (`<scope>__code`, `pattern__pbi_escaped`, …) themselves obey rule-of-two — don't create one unless reused across ≥2 models.

✅ Case 2 — suffixed-only concept, `_description` as anchor:
```yaml
# model description
description: |
  `<concept>_code` columns, anchored on `<concept>_description`: {{ doc('prismg2__code') }}
  `<concept>_description_pbi_escaped` columns, anchored on `<concept>_description`: {{ doc('pattern__pbi_escaped') }}

columns:
  - name: phase_description
    description: "{{ doc('prismg2__phase') }} {{ doc('prismg2__description') }}"
  - name: phase_code           # empty — hoisted
  - name: phase_description_pbi_escaped  # empty — hoisted
```

✅ Case 1 — bare concept present:
```yaml
- name: workspace_id
  description: "{{ doc('prismg2__workspace_id') }}"
```

❌ Concept doc attached on every form col:
```yaml
- name: phase_code
  description: "{{ doc('prismg2__phase') }} {{ doc('prismg2__code') }}"
- name: phase_description
  description: "{{ doc('prismg2__phase') }} {{ doc('prismg2__description') }}"
- name: phase_description_pbi_escaped
  description: "{{ doc('prismg2__phase') }} {{ doc('pattern__pbi_escaped') }}"
```
Why wrong: concept is named once per family, on the anchor only. Rest hoist.

❌ Fused concept+form doc block:
```sql
{% docs prismg2__phase_code %}Prism G2 code for the Phase concept.{% enddocs %}
```
Why wrong: kills reuse of `phase` on `_description` and of `_code` across other concepts.

**Parameterised Docs:**
When description needs shared body + col-specific reference, wrap `doc()` in a macro returning markdown. Place macro under `macros/docs/`. Macro name signals the pattern (greppable).
  - ❌ Hand-copy col name into 25 inline descriptions.
  - ✅ `description: "{{ sort_order_doc('phase_description_pbi_escaped') }}"`

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

**Shared Concepts (Pass-Through & Renames):**
- As soon as a concept appears in 2+ models — same name (pass-through) or renamed — establish a `doc()` block and reference it from every model: `description: "{{ doc('<scope>__<term>') }}"`. Single-model concepts stay inline.
  - ✅ `description: "{{ doc('prismg2__workspace_id') }}"` on both `project_id` (source) and `workspace_id` (downstream).

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
- [ ] **Errors:** Are errors preserved as data (no silent drops)? Do error reasons use `lower_snake_case`? Do Marts filter errors out unless generating an error mart?
- [ ] **YAML Tone:** Does YAML use block scalars (`|`) and document the business contract rather than SQL mechanics?
- [ ] **YAML Rules:** Are there zero hard-coded values in docs? Are referenced identifiers wrapped in backticks? Is every concept used in 2+ models wired through a shared `doc()` block?
- [ ] **Docs Scope:** Within-model repetition hoisted to model `description`? Cross-model concepts in doc blocks with correct scope prefix (`<source>__`, `<domain>__`, `conformed__`, `pattern__`) and correct file layout? Domain-scoped patterns kept narrow until a second consumer arrives?
- [ ] **Contracts:** Is the Grain and Error Grain declared in the `.yml`? Does the first line of each CTE comment properly declare and match the YAML grain?
- [ ] **Testing:** Do Fact tables (`fct_*`) have a surrogate key as the first column created from natural keys? Do Facts test `relationships` to dims? Are test configurations correctly nested under `arguments`?
- [ ] **Kimball:** Does the model design strictly follow Kimball dimensional modeling best practices?

Only output code if it strict-passes these constraints.
<!-- /process -->
