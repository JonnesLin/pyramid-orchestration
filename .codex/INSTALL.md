# Installing Pyramid Orchestration for Codex

## Prerequisites

Pyramid Orchestration requires main-session agent dispatch. In many Codex
setups, that means `multi_agent = true` must be enabled.

## Installation

### macOS / Linux

```bash
git clone https://github.com/JonnesLin/pyramid-orchestration.git ~/.codex/pyramid-orchestration

mkdir -p ~/.agents/skills
ln -s ~/.codex/pyramid-orchestration/skills/pyramid-orchestration ~/.agents/skills/pyramid-orchestration
```

### Windows

```cmd
git clone https://github.com/JonnesLin/pyramid-orchestration.git %USERPROFILE%\.codex\pyramid-orchestration

mkdir %USERPROFILE%\.agents\skills
mklink /J %USERPROFILE%\.agents\skills\pyramid-orchestration %USERPROFILE%\.codex\pyramid-orchestration\skills\pyramid-orchestration
```

## Updating

```bash
cd ~/.codex/pyramid-orchestration
git pull
```

Restart Codex after updating.

## Tool Mapping

| Pyramid references | Codex equivalent |
|---|---|
| Agent tool (Worker or child dispatch) | `spawn_agent` |
| Read | `read_file` |
| Write | `write_file` |
| Edit | `apply_diff` |
| Bash | `shell` |
| Grep | `grep_search` |
| Glob | `glob` |

## Notes

- Main dispatches Workers and child agents via `spawn_agent` in Codex
- Workers request deeper delegated work through files; they do not spawn agents
  directly
- Ensure your Codex config allows main-session agent dispatch
- The `/pyramid` command may need to be invoked as a skill reference rather
  than a slash command
