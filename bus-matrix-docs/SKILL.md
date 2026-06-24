---
name: bus-matrix-docs
description: >
  What a `bus-matrix.md` must hold for a dimensional data pipeline — the structural map of facts,
  grain, conformed dimensions, sources, and lineage, nothing else.
  Triggers when user mentions "bus matrix", "bus_matrix.md", "kimball matrix", "conformed dimensions",
  "document the bus matrix", or "bus matrix docs skill".
---

**Role:** Strict dimensional modeller. A bus matrix is the structural index of a pipeline — which
facts exist, their grain, which conformed dimensions they share, where their data enters, how it
flows. **Structure only.** No rationale, no narrative, no member enumeration. One `bus-matrix.md` per
context, beside the models it maps.

Follow the `docs-discipline` skill — lean, caveman-lite, greenfield. Follow `pipeline-layers` for
layer names.

## Canonical skeleton — reproduce this shape exactly

Headings and table columns are fixed. Do not rename, reorder, add, or drop sections. Omit an
optional section only when empty.

```md
# <Context> Bus Matrix

<Intro: 3–5 sentences. Model scope; one clause per business process (fact); link CONTEXT.md
and the relevant ADRs. No rationale.>

## Bus matrix

| Business process (fact) | Grain | <conformed dim> | <conformed dim> | … |
|---|---|---|---|---|

## Dimensions

| Dimension | Grain | Key | Type | Notes |
|---|---|---|---|---|

## Facts

| Fact | Grain | Surrogate key | Measures | Degenerates |
|---|---|---|---|---|

## Sources

| Source | Origin |
|---|---|

## Connections

| Id | Folder | From | To | How used |
|---|---|---|---|---|

## Lineage

<one mermaid flowchart — see Lineage rules>

## Structural notes        ← optional

<only table-inexpressible structural rules; each ≤2 lines, ADR-linked, no why>
```

## Per-section rules

### Bus matrix table
- Fact rows × conformed-dimension columns. Cells: `✓`, `—`, or a deferred/indirect marker
  (`↗ via project`, `(future)`).
- Conformed dimensions only. A local (non-conformed) dimension is **not** a column.

### Dimensions table
- `Type` = the dimension's structural kind, one token (SCD1/2/3, spine, seed, static,
  generated). Structural kind only — conformance is a separate axis and lives in Notes, never
  here (a dimension is *both* a kind *and* conformed-or-local).
- **Notes allow-list** — only these structural Kimball facts belong:
  - conformance word (conformed / local-to-fact)
  - special member key(s) — reserved rows for absent / inapplicable / unresolved facts (e.g. Unknown `-1`)
  - hierarchy note (flattened / snowflaked / self-ref) — structural only
  - deferred status (FK minted, dimension not built yet)
  - durable natural key (e.g. natural id = key)
  - role-playing (one physical dimension played under several roles)
  - special-type marker: junk / mini-dimension / outrigger / bridge (multi-valued)
  - ADR link(s)
- **Notes forbid-list** — these move out, do not restate:
  - source paths, sheets, systems → **Sources / Connections**
  - M mechanics ("view over spine", "Table.FromRecords seed") → nowhere, it is just the model
  - attribute / column listings → the model `*.yml`
  - member enumeration, definitions → **CONTEXT.md**
  - rationale / why → **ADR**

### Facts table
- **Surrogate key = name only.** No `= a|b|c` formula — the grain column already implies
  composition. A genuinely non-obvious derivation goes to **Structural notes**, not the cell.
- **Degenerates = true degenerates only** (keys with no dimension). A dimension FK (e.g.
  `control_key`, `purpose_key`) is **not** a degenerate — its membership is already the `✓` in the
  bus matrix table. Do not list it here.
- `Measures` = additive/semi-additive measures by name.

### Sources table
- One row per pbiflow source (the `source('x')` handle), even if it feeds several `src_` models
  (the fan-out lives in Connections).
- `Origin` mirrors `pbiflow.yml` config terse — `kind · site/server · path-or-query-gist`. Never a
  second source of truth. Truncate any single config value over ~20 chars with `…`.

### Connections table
- **A row exists iff one of two cases holds. Nothing else gets a row.**
  1. **External ingestion** (a `source` → its `src_` model): `How used` = a 1–2 line human-readable
     note (where it lands + how the model uses it), aligned with the `src_*.yml` description.
  2. **Genuinely surprising** internal edge: `How used` = an **ADR ref only**, no prose.
- Self-explanatory structure (unions, folds, normal layer flow visible in the lineage) gets **no
  row** — every Connections row must tell the reader something the diagram cannot.
- `Folder` = the consumer's model folder (`00_src`, `04_pres`, …); group/sort rows by it.
- `From` / `To` = model (or source) names. **Pattern matching allowed** to collapse many edges into
  one row, e.g. `dimension_*` → `pres_dimension_*`.
- `Id` (`C1`, `C2`, …) optional — assign one only when the edge is worth referencing from the
  diagram. A **pattern row carries one id** that **all** its matching diagram edges share.

### Lineage
- **One** mermaid `flowchart LR`. Never multiple diagrams.
- Group by **business process**, not by layer — one subgraph per process (e.g. Conformed
  dimensions / Capital / School / Office / Control / Consolidated), plus a `Sources` subgraph for
  external systems. Layer order is conveyed by flow direction and `src_/stg_/int_/…` prefixes.
- Conformed-dimension nodes fan out to the process boxes they serve — that fan-out *is* the
  conformance evidence.
- Edge labels: **only** the `C#` of a called-out Connections row. No prose on edges.
- **Complete** — every model in the pipeline appears as a node, exactly once. A model in the
  code but not the diagram is drift.
- Every conformed key in the matrix appears in the lineage; a minted FK with no dimension yet still
  shows, marked deferred.

### Structural notes (optional)
- Only structural rules a table cell cannot hold: composite-grain unpivot/collapse, double-entry
  contra netting, anchor rules, non-obvious key derivation.
- Each ≤2 lines, ADR-linked, **no why** (the ADR holds the why). If the tables are plain, write
  nothing.

## Delegate — stays out of the matrix

- **Terms, members, definitions → CONTEXT.md.** Name entities here; define them there. Never
  enumerate members (see `docs-discipline` Rule 4).
- **Why / rationale → ADR.** The matrix states structure and links the ADR; it never argues a
  choice.
- **Column descriptions, tests → model `*.yml`.** Reference by name; never restate.

## Self-check — run before finishing

Reject the doc if any is true:

- [ ] A heading or table column differs from the canonical skeleton.
- [ ] More than one mermaid diagram, or the diagram is grouped by layer not process.
- [ ] A model in the pipeline is missing from the lineage (drift against the code).
- [ ] A Dimensions `Notes` cell holds anything off the allow-list (source path, M mechanics,
      column list, member list, why).
- [ ] A Facts `Surrogate key` cell holds a `= …` formula, or a `Degenerates` cell lists a
      dimension FK.
- [ ] A Connections row is neither external ingestion nor surprising-with-ADR (it restates the
      diagram).
- [ ] An `Origin` value over ~20 chars is not truncated.
- [ ] Any prose answers *why* (belongs in an ADR) or enumerates members (belongs in CONTEXT.md).
- [ ] The matrix was changed without updating the lineage in the same edit (a stale map misleads).

## Worked minimal example

```md
# Orders Bus Matrix

Dimensional model for the order pipeline. Two business processes: order **placement**
(`fact_order_line`) and order **fulfilment** (`fact_shipment`). Terms in [CONTEXT.md](../CONTEXT.md).

## Bus matrix

| Business process (fact) | Grain | `dimension_date` | `dimension_customer` | `dimension_product` |
|---|---|---|---|---|
| Placement — `fact_order_line` | order × line | ✓ | ✓ | ✓ |
| Fulfilment — `fact_shipment` | shipment | ✓ | ✓ | — |

## Dimensions

| Dimension | Grain | Key | Type | Notes |
|---|---|---|---|---|
| `dimension_date` | one day | `date_key` | generated | conformed; unknown `0` |
| `dimension_customer` | one customer | `customer_key` | SCD2 | conformed; unknown `-1` |
| `dimension_product` | one SKU | `product_key` | SCD1 | unknown `-1` |

## Facts

| Fact | Grain | Surrogate key | Measures | Degenerates |
|---|---|---|---|---|
| `fact_order_line` | order × line | `order_line_key` | `quantity`, `amount` | `order_id`, `line_no` |
| `fact_shipment` | shipment | `shipment_key` | `parcel_count` | `tracking_no` |

## Sources

| Source | Origin |
|---|---|
| `orders_api` | rest · `api.shop…` · `/orders` |
| `wms_export` | sftp · `wms-host` · `ship/*.csv` |

## Connections

| Id | Folder | From | To | How used |
|---|---|---|---|---|
| C1 | `00_src` | `orders_api` | `src_order_line` | One row per order line; paging flattened. |
| C2 | `00_src` | `wms_export` | `src_shipment` | Nightly parcel manifest, one row per parcel. |
|    | `04_pres` | `dimension_*` | `pres_dimension_*` | ADR-0009 |

## Lineage

flowchart LR
  subgraph sources
    orders_api
    wms_export
  end
  subgraph conformed["Conformed dimensions"]
    dimension_date
    dimension_customer
    dimension_product
  end
  subgraph placement["Placement"]
    src_order_line --> stg_order_line --> int_order_line --> fact_order_line
  end
  subgraph fulfilment["Fulfilment"]
    src_shipment --> stg_shipment --> fact_shipment
  end
  orders_api -- C1 --> src_order_line
  wms_export -- C2 --> src_shipment
  dimension_customer --> fact_order_line
  dimension_customer --> fact_shipment
```

## When to use

- Adding or reshaping a fact, dimension, source, or ingestion edge.
- Onboarding to a pipeline.
- Checking whether a new model conforms to existing dimensions.

## Style

Write the `bus-matrix.md` in `docs-discipline` style. If you have not read the
`docs-discipline` skill this session, read it now with the `read_file` tool before
writing. Do not paraphrase the rules from memory — the skill is the source of truth.
