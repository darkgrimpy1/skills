---
name: pipeline-layers
description: >
  Layer responsibility rules for any data pipeline. Defines purpose and
  boundaries of `src`, `stg`, `int`, `mart`. Engine-agnostic (SQL/dbt, M/pbiflow,
  Spark, dlt). Triggers on "pipeline layers", "src/stg/int/mart layer",
  "layer responsibility", "data modelling layers".
---

**Role:** Strict data modeller. Enforce layer boundaries. Engine-agnostic.

## Layers

```
src  →  stg  →  int  →  mart
raw     typed   logic   consumable
```

One model = one file. Ext follows engine. Naming: `<layer>_<entity>[_<qualifier>]`, snake_case.

## `src` — Source

**Purpose:** Land external bytes as a table. Nothing else.

**Allowed:** connector call · structural filter (path, ext, filename, sheet) · format parse to table · header promotion · union by column name across files (missing cols → null) · retain connector-supplied metadata as-is (e.g. `_source_file`, source modified date).

**Drift policy:** `src` does not police schema drift. New/missing/reordered columns flow through. Detection and response belong downstream (`stg` contract, tests).

**Forbidden:** type cast · rename · drop · reorder · currency/date/number/bool parse · derived cols (snapshot date, keys, flags, hashes) · business-rule row filter · refs to other models · value-transforming UDFs.

**Why:** faithful mirror. Debug = `src` vs raw export. Cleaning hides provenance.

**Schema contract:** every column listed, default type.

**Test:** swap source for hand-saved raw export → output identical post format-parse.

## `stg` — Staging

**Purpose:** Make `src` usable. Rename, cast, and apply transformations that will **never need to change**. Treat `stg` like a streaming table — append-only in spirit, schema must be stable. You can't retroactively re-cast a streaming column; same discipline here.

**1:1 with `src`:** every `src_X` has exactly one `stg_X`. Same suffix, same grain, same row count.

**Allowed:** rename to `lower_snake_case` · type cast · column selection (subset) · stable, value-preserving transforms (e.g. strip currency symbols, parse dates) · rename source-metadata cols.

**Forbidden:** row filtering · joins · refs to other `stg` · business-rule logic that may evolve · derived/calculated columns · aggregations · deduplication.

**Why:** downstream depends on `stg` schema as a contract. Anything volatile belongs in `int`.

**Test:** row count == `src` row count. Schema stable across runs.

## `int` / `mart`

TBD.
