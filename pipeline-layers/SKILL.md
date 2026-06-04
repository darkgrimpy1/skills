---
name: pipeline-layers
description: >
  Layer responsibility rules for any data pipeline. Defines purpose and
  boundaries of `src`, `stg`, `int`, `mart`/`model`, `pres`. Engine-agnostic (SQL/dbt, M/pbiflow,
  Spark, dlt). Triggers on "pipeline layers", "src/stg/int/mart layer",
  "layer responsibility", "data modelling layers".
---

**Role:** Strict data modeller. Enforce layer boundaries. Engine-agnostic.

## Layers

```
src  →  stg  →  int  →  mart  →  pres
raw     typed   logic   consumable  user-facing
```

One model = one file. Ext follows engine. Naming: `<layer>_<entity>[_<qualifier>]`, snake_case.

**Layer name aliases:** `mart` and `model` interchangeable (Kimball consumable layer). Pick one per repo, stay consistent.

**Presentation layer:** folder word is `presentation` (or `pres`); file suffix is `pres`. Pure rename layer — friendly column names, no table renames, no new modelling logic.

**Layer order prefix:** numeric prefix (`00_`, `01_`, `02_`…) is inferred from folder structure, not hard-coded. Read the model folders, sort, assign order. `src`=lowest. Don't assume a fixed number per layer.

## `src` — Source

**Purpose:** Land external bytes as a table. Nothing else.

**Allowed:** connector call · structural filter (path, ext, filename, sheet) · format parse to table · header promotion · union by column name across files (missing cols → null) · retain connector-supplied metadata as-is (e.g. `_source_file`, source modified date).

**Drift policy:** `src` does not police schema drift. New/missing/reordered columns flow through. Detection and response belong downstream (`stg` contract, tests).

**Forbidden:** semantic type cast (date/number/bool/currency parse) · rename · drop · reorder · derived cols (snapshot date, keys, flags, hashes) · business-rule row filter · refs to other models · value-transforming UDFs.

**Deterministic-types exception:** Some engines must know every column type at deploy (dataflow/TMSL emit, streaming sinks). When so, cast **everything to string** — uniform, zero interpretation, still a faithful mirror. All-string is the *only* cast `src` may make; never date/number/bool here — that stays `stg`'s job. Then declare every column `string` in the schema contract.

**Why:** faithful mirror. Debug = `src` vs raw export. Cleaning hides provenance.

**Schema contract:** every column listed. Default type — or `string` everywhere under the deterministic-types exception.

**Test:** swap source for hand-saved raw export → output identical post format-parse.

## `stg` — Staging

**Purpose:** Make `src` usable. Rename, cast, and apply transformations that will **never need to change**. Treat `stg` like a streaming table — append-only in spirit, schema must be stable. You can't retroactively re-cast a streaming column; same discipline here.

**1:1 with `src`:** every `src_X` has exactly one `stg_X`. Same suffix, same grain, same row count.

**Allowed:** rename to `lower_snake_case` · type cast · column selection (subset) · stable, value-preserving transforms (e.g. strip currency symbols, parse dates) · rename source-metadata cols.

**Forbidden:** joins · refs to other `stg` · business-rule logic that may evolve · derived/calculated columns · aggregations · deduplication.

**Row filtering (conditional):** Allowed **only** when the excluded rows are *permanently* out of scope and you are absolutely certain no future requirement will need them back — e.g. a fixed test/UAT domain, or SCD2 non-current rows. If there is any chance the rows return to scope, filter in `int` instead. When in doubt, do not filter here.

**Why:** downstream depends on `stg` schema as a contract. Anything volatile belongs in `int`.

**Test:** row count == `src` row count. Schema stable across runs.

## `int` — Intermediate

**Purpose:** All reshaping, conforming, and **key minting**. The only layer where cross-entity joins and business logic live. Produces reusable building blocks (conformed spines, mapping tables) for the consumable layer.

**Allowed:** join · unpivot/pivot · union · distinct · dedup · survivorship (alias collapse) · cross-join · calendar/spine generation · business-rule row filter · derived cols · **mint surrogate keys** (sequence/index) for entities whose key is not computable from a natural key.

**May reference:** `src`, `stg`, `seed`, other `int`. **Never** reads `model`/`mart` or `pres` (no reading down a layer — that's why spines belong here, not in `model`).

**Why:** one place to look for logic; keeps `model`/`pres` thin and `stg` stable.

## `mart` / `model` — Consumable

**Purpose:** Assemble Kimball dimensions and facts from `int_*` / `seed_*`. Thin shaping only — select, rename to physical names, resolve FKs. No new reshaping.

**Naming:** `dimension_<entity>` / `fact_<entity>` (full word `dimension`, not `dim`).

**Allowed:** select/rename · FK resolution · `is_*` flags trivially derived from present cols.

**Reference rule (strict):**
- Reads `int_*` / `seed_*` (upstream) and functions.
- **Never references a sibling `model`/`mart`** — no dim→dim, no fact→dim, no fact→fact. Anything a sibling would provide (e.g. a surrogate key) must be minted upstream in `int` and joined from there.
- **May read `stg` directly only** when an `int` model would be a pure, trivial, single-use passthrough. If the table is real computation or a candidate conformed spine, build it in `int` instead (because `int` can't read `model`).

## `pres` — Presentation

**Purpose:** Pure rename to user-facing labels. The semantic/report layer's friendly face.

**Naming:** `pres_<entity>` (e.g. `pres_dimension_risk`), 1:1 with its `model` parent.

**Allowed:** rename columns to `Title Case` friendly names **only** — including FK keys (`risk_owner_key` → `"Risk Owner Key"`).

**Forbidden:** any logic · row add/drop · new/derived columns · **renaming the table/model itself** · referencing anything other than its single `model` parent.

**Why:** report authors bind to stable friendly names; all modelling stays in `model`/`int`.
