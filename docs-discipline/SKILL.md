---
name: docs-discipline
description: Discipline for writing comments, docstrings, READMEs, ADRs, and prose docs. Consult before creating or editing any documentation, and when editing code that already has comments or docstrings. Read the skill for the rules.
---

Write documentation lean. Code say *how*. Comment say *why*. Default = no comment.

This skill governs comments, docstrings, README, CONTEXT.md, ADRs, any prose doc. Consult it before writing or editing documentation. Does not govern commit messages or PR text.

## When to Use

Use this skill when you:

- Write or edit any comment, docstring, or prose doc.
- Touch code that already has documentation.
- Edit a code region containing doc violations — fix the surrounding docs too, even if not asked.

## Rule 1 — Write greenfield (no diff or provenance narration)

**Diff narration** is a comment that narrates the change instead of describing current state. It rots the instant the diff lands.

Comment describe what *is*, never what *changed*. Change story belong in commit message or PR, not code. (Literature calls this a "journal comment", from *Clean Code*.)

Kill these — each wraps a legitimate *why* in change story:

```python
# Now we validate the token before parse — parser trusts claims blindly
# Changed to a set: duplicate SKUs double-counted with a list
# No longer retries on 429: the gateway already retries
```

Keep the why, drop the change story:

```python
# Token must be validated before parse — parser trusts claims blindly
# Set, not list: duplicate SKUs would double-count
# No retry on 429: the gateway already retries
```

When stripping the change story leaves only what the code already says (`# Updated to handle the null case` → "handles the null case"), no comment survives: delete it.

Mechanical check: these words in a comment almost always mean history, not code. *Now, previously, changed, updated, moved, renamed, new, no longer, refactored, used to.* Runtime time ("now" as execution time, not writing time) is exempt from the word check. But passing greenfield is necessary, not sufficient: the other rules still apply, and they kill most temporal comments.

```python
# ✗ expires 24h from now                deictic: hides the anchor event
# ✗ expires 24h after issuance          greenfield-clean, but Rules 2 & 4 kill it:
#                                       the value belongs in a constant, the anchor in a name
# ✓ expires_at = issued_at + TOKEN_TTL
```

The runtime-time comment that survives every rule names *which moment* anchors a calculation, a business rule the code cannot express:

```python
# Priced as-of query time, not order time: quotes must show the market the
# customer sees (FIN-231). Looks stale-prone; is deliberate.
```

**Greenfield.** Diff narration is the *temporal* case of a wider rule: describe the thing as if built from scratch, in its current state. The *spatial* case is **provenance narration** — narrating where the thing came from, what it replaces, or upstream lineage/mechanics, as if that were its description. It rots and distracts from what the thing *is*, just like a diff.

Kill these:

```python
# Replaces the legacy interest calc        ← provenance
# Ported from the Access version           ← provenance
# Reads the P&L legs of the upstream join  ← lineage mechanics as description
```

This binds **descriptive** docs — README, CONTEXT.md, schema/yml descriptions, docstrings: say what the thing *is* and *means*, not how it was wired or what it succeeded. (Explaining *why* a choice was made is still allowed — that's rationale, see below — provenance is narrating the history, not the reason.) Rationale is written absolutely, never comparatively: "context avoids Redux ceremony for read-heavy state", not "switched away from Redux because...".

**Carve-out — comparison docs.** When the document's *purpose* is comparison — a changelog, a migration guide, an ADR's Considered Options, a deliberate before/after — provenance and diffs are the whole point. Write them. The greenfield rule binds descriptive docs, not comparative ones.

Example — an ADR whose contract *requires* it to flag what it supersedes and justify why:

```md
# Source the account dimension from finance master

Status: supersedes ADR-0014.

ADR-0014 drew the account dimension from the template `Mapping` sheet; we now
source it from the finance master, because the template drifted out of sync. That
provenance is the decision's substance — name it.
```

Here "supersedes ADR-0014" and "drew from … now source from" are provenance on purpose: the doc exists to record a reversal. Greenfield would gut it. Contrast a *descriptive* doc — the dimension's yml — which says only what the account dimension *is* today, never that it once came from the template.

## Rule 2 — Make code self-documenting first

Before you add a comment, try to delete the need for it. A comment is the last resort, not the first.

When tempted to comment, do this first:

1. **Rename** — better variable/function name often replace whole comment.
2. **Extract constant** — magic number/string become a named constant.
3. **Extract function** — name the block by extracting it; the name is the comment.
4. **Simplify** — restructure so intent is obvious.

Only after self-documenting fails, write the comment.

```python
# Bad: magic number + explanatory comment
time.sleep(300)  # wait 5 minutes for replication

# Good: self-documenting, no comment needed
REPLICATION_LAG = timedelta(minutes=5)
time.sleep(REPLICATION_LAG.total_seconds())
```

## Rule 3 — Right altitude

Explain **block or function intent**, not every line. Self-documenting code does not need narration.

- Do not comment every line.
- Do not even comment every block — only the ones with non-obvious intent.
- One comment that frame a whole function beats five comments narrating its lines.
- If the block is already clear, add nothing.

```python
# Bad: line-by-line narration of obvious code
def total(items):
    s = 0              # init sum
    for i in items:    # loop items
        s += i.price   # add price
    return s           # return total

# Good: clear code, zero comments
def total(items):
    return sum(item.price for item in items)
```

## Rule 4 — Describe the axis, don't enumerate volatile members

Docs must survive membership drift. Before listing members, ask: closed and definitional, or open and volatile?

- **Closed, definitional** — the full list *is* the meaning. List it all.
- **Open or volatile** — describe what the axis *means*, not its current members. Drop an example only if it earns understanding; never an exhaustive list.
- **Dependent / derived term** — when a term's meaning is a predicate over another enum (`Active` = pre-decision statuses), define it by the *criterion*, not the member list, so new members self-classify. Keep the authoritative members in one place (seed/dimension/enum) and reference it; never duplicate the list into prose.

```text
✓ movement_type — whether a row is a dated flow or a period-opening anchor.
✓ Account type — P&L or Balance Sheet; the closed pair is the definition.
✗ movement_type — Transaction, Opening Balance, …          (rots on the next member)
✗ Account type — P&L, …                                    (truncating a closed set loses the definition)

✓ Active — a not-yet-resolved status; canonical members live in dimension_status.
✗ Active — Under-Review or Pending     (rots on a new pre-decision status, and forks the source of truth)
```

A bare enumeration of a volatile set is diff-narration waiting to happen — correct only until the data changes, then it silently lies.

Same rule at member-count one: never restate a constant's value in prose. Reference the name; the value lives in one place.

```python
# ✗ Returns true if not updated in the last 5 minutes   (rots when STALE_THRESHOLD changes)
# ✓ Returns true if older than STALE_THRESHOLD
```

Carve-out: unit translation of an adjacent magic literal is clarity, not duplication. `1048576  # 1 MB` is fine.

## When a comment IS warranted

Add a comment only when it carries information the code cannot:

- **Why, not what** — non-obvious reason, business rule, tradeoff.
- **Side-effect warning** — flag a non-obvious side effect: mutation, ordering dependency, I/O, global state. Scope strictly to *side effects*, not general "be careful" notes and not error conditions.
- **Intent over mechanics** — meaning of a regex, bitmask, or magic constant that survived self-documenting.
- **Surprising "why not"** — code that looks like a bug but is correct. A rejected *design* belongs in an ADR, not a comment; if the repo documents ADRs, write the ADR and link it from the comment.
- **External reference** — link a spec, ticket, RFC, or ADR.
- **TODO / FIXME** — flagged debt, with context (ticket, owner, reason). Bare `TODO` is useless.
- **Expiry condition** — every workaround, hack, or version-pinned branch states the condition under which it dies: "remove when Safari 14 support drops", with the bug link. Prefer a condition the reader can test over a calendar date; use a date only when the trigger genuinely is one (API sunset, license expiry). A workaround with no removal condition is permanent code in disguise. Forward-looking, so greenfield permits it. Carry it on a `TODO`/`HACK` marker so it is greppable — an obligation buried in prose is never found. Markers are for *obligations* only; a comment describing runtime behavior ("priced as-of query time") is plain description, no marker.
- **Public API docstring** — the contract callers read instead of the body. Allowed even when "obvious", but do not restate the signature or types. Say what the signature cannot: behavior at the edges, error semantics, units, invariants the caller must hold.

```python
def shipping_cost(weight_kg: float, zip_code: str) -> Decimal:
    """Quote for shipping one parcel; weight tiers live in PRICING_TIERS.

    Returns 0 for destinations we don't ship to instead of raising.
    Call can_ship_to() first to tell "free shipping" from "can't ship".
    """
```

## Anti-patterns — delete on sight

- Diff narration (Rule 1).
- Provenance narration in descriptive docs — "replaces X", "ported from Y", upstream lineage as description (Rule 1).
- Restating what the code plainly says.
- Commented-out code, and headstones over deleted code ("used to retry here") — git remembers; delete both.
- Noise comments (`# constructor`, `# getter`).
- Docstrings that just echo the signature and types.
- Bare `TODO` with no context.

## Comment rot

A stale comment is worse than no comment. Keep each comment next to the code it explains and update both together. Diff narration is the worst rot — it is stale the moment it is written.

Editing code near a stale comment: fix the comment or delete it, never append a correction ("actually this also handles X now"). One comment, one truth, present tense.
