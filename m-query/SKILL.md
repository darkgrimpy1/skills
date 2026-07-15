---
name: m-query
description: >
  Power Query M conventions for pbiflow models. Triggers when user mentions
  "m-query", "power query", "m language", ".m model", "powerquery", or edits
  `**/*.m` files. Read before writing or reviewing M code.
---

**Role:** Power Query M engineer. Write M for performance and clarity.

Lightweight skill. Grows over time.

## Rules

### 1 — Join, do not iterate

Match rows with a join. Never scan one table per row of another.

- ❌ `Table.AddColumn(t, "x", each List.Contains(other[id], [id]))` — O(n*m) scan.
- ❌ `Table.SelectRows(t, each List.Contains(keep[id], [id]))` — same scan.
- ✅ `Table.Join` / `Table.NestedJoin` on the key — hash join, O(n+m).

Filter-by-membership is an inner/anti join, not a `List.Contains` loop.

### 2 — Expand after join, never operate on nested columns

After `Table.NestedJoin`, expand the columns you need. Do not reach into the
nested table per row with `each`.

- ❌ `Table.AddColumn(j, "v", each Table.Column([_j], "v"){0} otherwise null)`
  — runs a lookup per row, defeats the join.
- ✅ `Table.ExpandTableColumn(j, "_j", {"v"}, {"v"})` — set-based, cheap.

Coalesce a missing match after expand (`[v] ?? 0`), not inside the `each`.

### 3 — Narrow before expand

Reduce the column set *before* `Table.ExpandTableColumn` / `Table.ExpandListColumn`.
Expand copies every carried column onto each new row, so a wide input balloons the
buffer. Non-folding queries must do it — the local engine prunes nothing for you;
when folding, still worth it to nudge the source optimiser to select fewer columns.

- ❌ expand first, narrow after — wide columns duplicated, then dropped.
- ✅ `Table.SelectColumns(j, {"id", "_j"})` then `Table.ExpandTableColumn(...)`.
- ✅ `Table.RemoveColumns(j, {"notes", "raw"})` then `Table.ExpandTableColumn(...)`.
