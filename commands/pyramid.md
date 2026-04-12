---
name: pyramid
description: Execute a plan using hierarchical agent orchestration. Reduces main agent context usage by delegating through Worker Agents.
---

# /pyramid

Execute a task plan using pyramid orchestration.

## Usage

```
/pyramid [path-to-plan]
```

If no path is provided, look for a plan file in `docs/plans/` or prompt the user to specify one.

## Behavior

When this command is invoked, activate the `pyramid-orchestration` skill and follow its protocol:

1. Locate and read the plan file
2. Assess task count to determine orchestration mode
3. Group tasks and dispatch Workers
4. Collect summaries and report progress
5. Handle review and fix cycles

This command is the entry point. The skill contains the full orchestration logic.
