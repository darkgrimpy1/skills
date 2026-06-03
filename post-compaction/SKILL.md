---
name: post-compaction
description: Applies right after any conversation compaction or summarization event. The moment you detect context has been compacted or summarized, before doing other work, re-read the relevant instructions.md files with the read_file tool. Read the skill for the full recovery steps.
---

Compaction drops detail. Instructions are the first thing to restore.

## When to Use

Apply this skill immediately after any compaction or summarization event, before doing other work.

## What to Do

1. **First and foremost**, re-read the relevant `instructions.md` files with the `read_file` tool.
2. Do not rely on the compacted summary for instruction content — it is lossy.
3. Only after re-reading, resume the task.
