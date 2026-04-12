# Pyramid Orchestration for Codex

## Overview

Pyramid Orchestration adds hierarchical agent delegation to Codex. Instead of one agent managing all subagents directly, it introduces a Worker layer that absorbs detailed results and returns only summaries to the orchestrator.

## Requirements

- Codex with `multi_agent = true` enabled
- Git installed and configured

## Installation

See [.codex/INSTALL.md](../.codex/INSTALL.md) for step-by-step instructions.

## How Workers Map to Codex

In Codex, the Agent tool maps to `spawn_agent`. When the pyramid skill dispatches a Worker, it uses `spawn_agent` to create an independent agent with its own context.

The Worker:
1. Receives a task assignment via the spawn prompt
2. Uses available tools (read_file, write_file, shell, etc.) to execute
3. Can spawn its own sub-agents for further delegation
4. Returns a brief summary when complete

## Differences from Claude Code

| Feature | Claude Code | Codex |
|---------|------------|-------|
| Worker dispatch | Agent tool | spawn_agent |
| Parallel Workers | Native parallel tool calls | Sequential or manual parallel |
| Slash command | `/pyramid` | Invoke as skill reference |
| Worker isolation | Automatic context isolation | Automatic via spawn_agent |

## Limitations

- Codex may not support parallel agent spawning — Workers may execute sequentially
- The `/pyramid` slash command requires plugin command support; if unavailable, reference the skill directly
