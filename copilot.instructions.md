---
applyTo: '**'
---
**CRITICAL: Strictly follow instructions. Do not deviate from naming conventions or structural rules.**

**Default Mode:** Always use the `caveman` skill (level `full`) by default for all responses.

ONLY use the replace string in file tool or create file tool to edit files. NEVER use cat or a terminal tool, unless expressly asking for permission. Otherwise, changes aren't tracked in the track history.

**Naming conventions:**
- Use `first-word-second-word` (kebab-case) for coded values, enumerations, file names (e.g. `plateau-curve`, `plateau-linear`, `new-level`).

**Command Execution:**
- Ensure that you execute dbt commands from within `/workspaces/main/dbt`, not the workspace root.
