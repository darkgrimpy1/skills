---
name: bus-matrix-docs
description: >
  What a `bus_matrix.md` must hold for a dimensional data pipeline — the structural map of facts,
  grain, and conformed dimensions, nothing else.
  Triggers when user mentions "bus matrix", "bus_matrix.md", "kimball matrix", "conformed dimensions",
  "document the bus matrix", or "bus matrix docs skill".
---

**Role:** Strict dimensional modeller. A bus matrix is the structural index of a pipeline — which
facts exist, their grain, which conformed dimensions they share. Structure only. One `bus_matrix.md`
per context, beside the models it maps.

## What goes in

Required sections, in order:

1. **Intro paragraph** — model scope, business processes (one per fact), source layer, links to
   CONTEXT.md and ADRs. Three to five sentences.
2. **Bus matrix table** — the core artefact. Fact rows × conformed-dimension columns. Mark each cell
   `✓`, `—`, or a deferred/indirect marker (e.g. `↗ via project`, `(future)`). Conformance at a
   glance.
3. **Dimensions table** — one row per dimension: grain, key (formula if derived), SCD/seed/spine
   type, structural notes (unknown member, flattened hierarchy, deferred). ADR link in the note.
4. **Facts table** — one row per fact: grain, surrogate/degenerate key, measures, flags.
5. **Structural subsections** — short subsections for structure a table cell cannot hold:
   composite-grain rules, key derivation, unpivot/collapse shape, conformance mechanics, deferred
   dimensions, unknown-member use. Each links the ADR that holds the rationale. Bus-specific
   groupings get their own subsection here too — e.g. presentation / report templates (downstream
   outputs that are not Kimball facts). Different busses carry different subsections; omit any area
   the tables already make plain.
6. **Lineage** — model dependency flow as a mermaid diagram. Layers per the `pipeline-layers` skill.

## Delegate — stays out

- **Terms → CONTEXT.md.** Name entities here; define them there. Intro links it. For the format,
  read `CONTEXT-FORMAT.md` from the `grill-with-docs` skill (`read_file` tool).
- **Decisions and rationale (the *why*) → ADR.** The matrix never argues a choice — it states the
  structure and links the ADR that justifies it (`single conformed rating dim — ADR-0001`). Offer an
  ADR only when hard to reverse, surprising, and a real trade-off. For the format, read
  `ADR-FORMAT.md` from the `grill-with-docs` skill (`read_file` tool).
- **Column descriptions, tests, model docs → dbt model YML.** Reference models and columns by name;
  never restate their YML descriptions or test coverage.

## Rules

- Structural only. State what links what and at what grain. No rationale prose — link the ADR.
- Reference models and columns by name; never duplicate model-YML docs or CONTEXT.md definitions.
- Every conformed key appears in both the matrix and the lineage. Mark deferred dimensions
  explicitly; a minted FK with no dimension yet still shows.
- A structural subsection holds structure, not justification. If the tables are plain, write nothing.
- Update the matrix and lineage in the same change as the model — a stale map misleads.
- Follow the `docs-discipline` skill — lean, caveman-lite.

## When to use

- Adding or reshaping a fact or dimension.
- Onboarding to a pipeline.
- Checking whether a new model conforms to existing dimensions.
