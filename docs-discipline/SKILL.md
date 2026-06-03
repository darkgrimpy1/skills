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

## Rule 1 — No diff narration

**Diff narration** is a comment that narrates the change instead of describing current state. It rots the instant the diff lands.

Comment describe what *is*, never what *changed*. Change story belong in commit message or PR, not code. (Literature calls this a "journal comment", from *Clean Code*.)

Kill these:

```python
# Now we validate the token before parsing   ← diff narration
# Changed to use a set for faster lookup      ← diff narration
# This used to call the old API               ← diff narration
# Updated to handle the null case             ← diff narration
```

Keep current-state framing only when it earns its place:

```python
# Token must be validated before parse — parser trusts claims blindly
```

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

## When a comment IS warranted

Add a comment only when it carries information the code cannot:

- **Why, not what** — non-obvious reason, business rule, tradeoff.
- **Side-effect warning** — flag a non-obvious side effect: mutation, ordering dependency, I/O, global state. Scope strictly to *side effects*, not general "be careful" notes and not error conditions.
- **Intent over mechanics** — meaning of a regex, bitmask, or magic constant that survived self-documenting.
- **Surprising "why not"** — code that looks like a bug but is correct. A rejected *design* belongs in an ADR, not a comment; if the repo documents ADRs, write the ADR and link it from the comment.
- **External reference** — link a spec, ticket, RFC, or ADR.
- **TODO / FIXME** — flagged debt, with context (ticket, owner, reason). Bare `TODO` is useless.
- **Public API docstring** — the contract callers read instead of the body. Allowed even when "obvious", but do not restate the signature or types.

## Anti-patterns — delete on sight

- Diff narration (Rule 1).
- Restating what the code plainly says.
- Commented-out code — git remembers; delete it.
- Noise comments (`# constructor`, `# getter`).
- Docstrings that just echo the signature and types.
- Bare `TODO` with no context.

## Comment rot

A stale comment is worse than no comment. Keep each comment next to the code it explains and update both together. Diff narration is the worst rot — it is stale the moment it is written.

## Style — caveman lite

Write all comments and docs in **caveman lite** style. If you have not already read the `caveman` skill this session, read it now with the `read_file` tool before writing, then apply the **lite** level. Do not paraphrase the rules from memory — the skill is the source of truth.

This covers inline comments, docstrings, README, CONTEXT.md, ADRs, and other prose docs. It does **not** cover commit messages or PR descriptions — those are out of scope for this skill.

```python
# Bad (filler): We should probably just make sure to check if the user is
# actually valid here before we go ahead and continue with the request.

# Good (caveman lite): Reject invalid user before processing — downstream
# trusts the user object.
```
