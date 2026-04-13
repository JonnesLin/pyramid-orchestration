# Pyramid Orchestration — Gemini CLI

## Compatibility Notice

The automated pyramid protocol requires the main session to dispatch
independent agents. Gemini CLI does not currently support agent dispatch.

This means:

- the automated file-backed pyramid is **not available** on Gemini CLI
- you can still read the skill for the orchestration concepts
- you can use the same `.pyramid/runs/<run-id>/` file structure manually across
  sessions

## Tool Mapping

| Pyramid references | Gemini equivalent |
|---|---|
| Read | `read_file` |
| Write | `write_file` |
| Edit | `replace` |
| Bash | `run_shell_command` |
| Grep | `grep_search` |
| Glob | `glob` |
| Agent tool (Worker or child dispatch) | Not supported |

## Manual Pyramid Workflow for Gemini

Since agent dispatch is unavailable, approximate the pyramid manually with
separate sessions and the same file-backed handoff protocol:

1. Session 1 acts as **Main**: create the plan, create
   `.pyramid/runs/<run-id>/`, group tasks, and write `manifest.md`
2. Session 2 acts as **Worker A**: execute direct work, or write request files
   under `requests/`, then stop
3. Session 3 acts as **Child A-01**: fulfill one request file and write a
   result file under `results/`
4. Session 4 acts as **Worker A continuation**: absorb the result file,
   continue the task slice, and write a summary under `summaries/`
5. Repeat for each Worker group, then run separate review and verification
   sessions

Each session reads the plan file, `manifest.md`, and git log to understand
progress. This achieves the same context isolation as the automated pyramid,
but with manual session management.
