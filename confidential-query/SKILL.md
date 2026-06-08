---
name: confidential-query
description: >
  Mandatory pre-flight gate for any read query against a data warehouse, database,
  API, file, or other live data source. Treat every query and its output as if the
  full text will be published on the public internet tomorrow. Forces a leak-scope
  review before execution, classifies output shapes, and provides masking patterns
  that preserve joinability without leaking identifiers. Applies regardless of tool
  (dbt, SQL CLI, notebooks, REPL, scripts) or domain.
---

**Role:** Confidentiality gate. Block leak before it happens.

**Operating assumption:** Every query and result will be published online tomorrow
under your name. Chat transcripts, logs, memory, screenshots all leak eventually.

## Rules

<!-- rule domain="pre-flight" -->
Before running ANY data-access query:

1. Write a **Leak Scope** block above the call.
2. If any field is **Unsafe**, either rewrite to a Safe form or ask the user to
   authorise explicitly. Silence = refusal. Approval is per-query, never carried.
3. After execution, do not paraphrase identifying rows into chat summary text.
<!-- /rule -->

## Leak Scope Block

```
Leak Scope
- Purpose: <one-line goal>
- Source: <table / endpoint / file>
- Output rows (cap): <limit>
- Output columns: <list>
- Identifiers exposed: <none | column-list>
- Quantitative values: <none | bucketed | raw>
- Temporal precision: <none | year | month | day | timestamp>
- Verdict: <Safe | Unsafe — rewrite | Unsafe — needs approval>
```

## Classification

<!-- rule domain="classification" -->
First classify the **grouping / selected column**:

- **Public taxonomy** — non-identifying categories shared across the domain
  (`source_type`, `scope_code`, `country`, `status`). Bucket label reveals nothing.
- **Identifier** — names a specific entity (`facility`, `well_id`, `customer_id`,
  `email`, lat/lon, IP). The label itself IS the leak. Cardinality irrelevant.
- **Quasi-identifier** — not unique alone, identifying in combination or via
  linkage to public data (`postcode + dob + gender`; `business_unit +
  asset_group` at low k; sub-day timestamps linkable to news/weather/shift
  rosters / market ticks). Coarsen to the lowest precision the question needs.

| Output shape | Verdict |
|--------------|---------|
| `count(*)`, `count(distinct)`, null-coverage, schema introspection | Safe |
| Aggregates with no group-by | Safe |
| Group-by on **public taxonomy**, all buckets `>= k` (default 5) | Safe |
| Group-by on **identifier**, any aggregate, any cardinality | **Unsafe** |
| Group-by on multi-col **quasi-identifier** | Unsafe — k-anon risk |
| Group-by on taxonomy with any bucket `< k` | Unsafe — small-cell |
| Raw row sample with identifier + dated measure | Unsafe |
| Identifier alone, no measure, no date | Borderline — prefer masked |
| Hashed or re-keyed identifier + raw measure | Safe-for-debug |
| Free-text / comment / description columns | Unsafe by default (may embed PII) |
<!-- /rule -->

## Joinable-Mask Patterns

<!-- rule domain="masking" -->
Prove a relationship without revealing cleartext.

**1. Salted hash** — preserves equi-joins within session. Salt stays out of chat.
```sql
md5(concat(:session_salt, identifier_col)) as id_hash
```

**2. Re-keyed surrogate** — stable within one result set, no cross-query join.
```sql
dense_rank() over (order by identifier_col) as entity_no
```

**3. Bucketed measure** — quantise so hash cannot be back-solved by value.
```sql
round(measure, -3) as measure_k,  date_trunc('month', d) as d_month
```

**4. Residual-only validation** — verify a formula by counting violations.
```sql
count(*) filter (where abs((a + b) - c) > 1e-6) as mismatches
```

**5. Schema / coverage probe** — answer "what's in the table" without rows.
```sql
select column_name, data_type from information_schema.columns where ...
select count(*), count(col), count(distinct col) from t
```

**6. Top-k suppression** — `having count(*) >= 5`.
<!-- /rule -->

## Governance Principles

<!-- rule domain="governance" -->
Assume the query is published tomorrow. Therefore:

1. **Least privilege.** Select only columns the question needs. No `select *`.
   Push filters deep. Aggregate before sampling. Mask before sampling.
2. **No identifiers in echoed WHERE clauses.** `where customer_id = 'ACME-12345'`
   leaks as much as selecting it. Parameterise or hash.
3. **No "let me peek" samples.** Curiosity is the top leak source. Replace with
   schema + cardinality + null-coverage probes.
4. **No raw rows to chat.** If row-level inspection is needed, write to a file
   the user controls and reference the path.
5. **Log only the Leak Scope, not the data.**
<!-- /rule -->

## When Row-Level Disclosure Is Unavoidable

<!-- rule domain="escalation" -->
1. State: "Row-level sample required to verify <X>; masked forms insufficient
   because <Y>." Propose minimum rows, columns, and masking.
2. Ask: "Approve sampling N rows with <these columns unmasked>? (yes/no)"
3. Execute only on explicit yes. Do not echo rows in summary afterwards.
<!-- /rule -->

## Pre-Execute Checklist

<!-- process step="verify" -->
- [ ] **Leak Scope** block written above the call.
- [ ] Verdict **Safe**, or user explicitly approved this turn.
- [ ] Row cap set via tool's limit mechanism (defer to relevant SQL skill on syntax).
- [ ] Identifiers masked or omitted unless authorised.
- [ ] Free-text columns excluded unless the task is text analysis.
<!-- /process -->
