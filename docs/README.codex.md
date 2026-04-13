# Pyramid Orchestration for Codex

## Overview

Pyramid Orchestration adds file-backed hierarchical delegation to Codex.
Instead of one agent managing all jobs directly, it introduces a Worker layer
that absorbs detailed results and returns only summaries to the orchestrator.
Main is the only agent that uses `spawn_agent`.

## Requirements

- Codex with main-session agent dispatch enabled
- Git installed and configured

In many Codex setups, main-session dispatch means `multi_agent = true`.

## Installation

See [.codex/INSTALL.md](../.codex/INSTALL.md) for step-by-step instructions.

## How Workers Map to Codex

In Codex, the Agent tool maps to `spawn_agent`. The file-backed pyramid
protocol uses it this way:

1. **Main dispatches a Worker** via `spawn_agent`
2. **Worker executes direct work** using available tools
3. **Worker writes request files** under
   `.pyramid/runs/<run-id>/requests/` when deeper delegated work is needed
4. **Main dispatches child agents** for those request files via `spawn_agent`
5. **Children write result files** under `.pyramid/runs/<run-id>/results/`
6. **Main dispatches continuation Workers** to absorb those result files and
   keep moving
7. **Workers return brief summaries** when complete or at durable handoff
   points

The important constraint is that Workers do **not** call `spawn_agent`
themselves in this protocol.

## Differences from Claude Code

| Feature | Claude Code | Codex |
|---------|-------------|-------|
| Worker dispatch | Agent tool | `spawn_agent` |
| Child dispatch | Agent tool | `spawn_agent` from Main only |
| Parallel Workers | Native parallel tool calls | Sequential or manual parallel |
| Slash command | `/pyramid` | Invoke as skill reference |
| Worker isolation | Automatic context isolation | Automatic via `spawn_agent` |

## Limitations

- This is not true nested agent spawning; Main must service the request queue
- Codex may not support parallel agent spawning — Workers and child jobs may
  execute sequentially
- The `/pyramid` slash command requires plugin command support; if unavailable,
  reference the skill directly
- If main-session dispatch is unavailable, fall back to the manual multi-session
  workflow
