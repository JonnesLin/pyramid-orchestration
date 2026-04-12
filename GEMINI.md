# Pyramid Orchestration — Gemini CLI

## Compatibility Notice

Pyramid Orchestration requires subagent/multi-agent support to function. The core pattern (Main → Worker → Sub) depends on dispatching independent agents.

Gemini CLI does not currently support subagent dispatch. This means:

- The full pyramid pattern is **not available** on Gemini CLI
- You can still read the skill for the orchestration concepts
- For Gemini CLI, consider a manual equivalent: split your work into separate sessions, each handling a chunk of the plan, and use files for handoff between sessions

## Tool Mapping

| Pyramid references | Gemini equivalent |
|---|---|
| Read | read_file |
| Write | write_file |
| Edit | replace |
| Bash | run_shell_command |
| Grep | grep_search |
| Glob | glob |
| Agent tool (Worker dispatch) | Not supported |

## Manual Pyramid Workflow for Gemini

Since subagents are unavailable, you can approximate the pyramid manually:

1. Session 1: Create the plan, group tasks, commit plan file
2. Session 2: Execute task group 1, commit results
3. Session 3: Execute task group 2, commit results
4. Session N: Review all changes, commit review
5. Session N+1: Fix issues from review

Each session reads the plan file and git log to understand progress. This achieves the same context isolation as the automated pyramid, but with manual session management.
