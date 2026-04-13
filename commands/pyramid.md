---
name: pyramid
description: Execute a plan using pyramid orchestration. Reduces main agent
  context usage by routing detailed work through Workers and file-backed
  handoffs.
---

# /pyramid

Execute a task plan using pyramid orchestration.

## Usage

```
/pyramid [path-to-plan]
```

If no path is provided, look for a plan file in `docs/plans/` or prompt the
user to specify one.

## Behavior

When this command is invoked, activate the `pyramid-orchestration` skill and
follow its protocol:

1. Locate and read the plan file
2. Assess task count to determine orchestration mode
3. Create or resume a run directory under `.pyramid/runs/`
4. Group tasks and dispatch Workers
5. Service request files by dispatching child agents from Main
6. Re-dispatch continuation Workers and collect summaries
7. Handle review, fix, and verification cycles

This command is the entry point. The skill contains the full orchestration
logic.
