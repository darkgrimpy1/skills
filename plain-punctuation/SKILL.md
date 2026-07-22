---
name: plain-punctuation
description: >
  Hard ban on em-dashes and on brackets/parentheses as prose asides, in all writing:
  responses, comments, docs, commit messages. Brackets are allowed only where syntax
  requires them. Use in every session; applies to all prose output.
---

Two punctuation habits are banned outright. Both are crutches for sentences that were
never finished: the em-dash bolts a second thought onto the first, and the parenthetical
whispers something the writer could not commit to saying. Write the sentence properly
instead.

## Em-dashes: never

Do not use the em-dash (—), the en-dash-as-pause (–), or the double hyphen (--) in prose.
There is no "sparingly". Rewrite instead:

- Two thoughts joined by a dash are two sentences. Use a period.
- A dash introducing an explanation is a colon's job.
- A dash setting off an aside means the aside goes: fold it into the sentence or cut it.

Hyphens in compound words (read-only, first-class) and ranges (lines 10-20, 2024-2026)
are ordinary hyphens and are fine.

## Brackets: syntax only

Parentheses and brackets are allowed only where a language, format, or convention
requires them:

- Code, function calls, signatures: `calculateShipping(weight, zip)`
- Markdown links: `[site.yml](ansible/site.yml)`
- Shell, regex, JSON, YAML, math
- Established notation: version strings, citations, legal names, `(a)`-style list markers
  in quoted standards text

Banned everywhere else, which means: no parenthetical asides in prose. "(see above)",
"(which is rare)", "(more on this later)", "(i.e. the parser)". Each of these is a
sentence that was not given its own place. If the aside matters, promote it to a
sentence. If it does not, delete it. Abbreviation introductions are the one prose
exception: "Analysis Services (AS)" on first use is fine.

## Test

Before sending any prose: search it for — and for ( . Every hit is either syntax from
the allowed list or a rewrite you owe.
