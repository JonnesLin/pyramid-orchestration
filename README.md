# Pyramid Orchestration

Hierarchical agent orchestration for AI coding agents.

## The Problem

AI agents using flat delegation hit context limits fast. One main agent dispatching 8+ subagents accumulates all their results — code, reviews, test output — into a single context window. The result: frequent compaction, information loss, degraded performance.

## The Solution

Add an intermediate **Worker** layer between the orchestrator and the executors:

```
Main Agent (Orchestrator)
  ├── summary ← Worker A ← full result ← Sub 1
  │                       ← full result ← Sub 2
  ├── summary ← Worker B ← full result ← Sub 3
  │                       ← full result ← Sub 4
  └── summary ← Worker C ← full result ← Sub 5
                          ← full result ← Sub 6
```

- **Subagents → Workers**: full results (no information loss where decisions happen)
- **Workers → Main**: summaries only (Main only needs status, not details)
- **Workers are disposable**: their context is released after completion

The main agent's context stays light. Detailed work lives in files and git commits.

## Context Savings

| Mode | 8 subagents | Main context |
|------|------------|--------------|
| Flat | All results to Main | ~29,000 tokens |
| Pyramid (3 Workers) | Summaries to Main | ~5,600 tokens |

~5x reduction. Workers handle ~11,000 tokens each, then release.

## When to Use

| Subagents needed | Recommendation |
|-----------------|----------------|
| 1-3 | Flat delegation (no overhead needed) |
| 4-8 | Pyramid with 2-3 Workers |
| 9+ | Pyramid with 3-5+ Workers |

## Installation

**Note:** Pyramid Orchestration requires subagent/multi-agent support. Platforms without subagent dispatch can use the manual workflow described in the Gemini CLI section.

### Claude Code

Clone and copy to your plugins directory:

```bash
git clone https://github.com/JonnesLin/pyramid-orchestration.git
mkdir -p ~/.claude/plugins/local
cp -r pyramid-orchestration ~/.claude/plugins/local/pyramid-orchestration
```

Or install just the skill (lighter, no slash command):
```bash
mkdir -p ~/.claude/skills
cp -r pyramid-orchestration/skills/pyramid-orchestration ~/.claude/skills/pyramid-orchestration
```

Restart Claude Code after installation.

### Cursor

In Cursor Agent chat:

```text
/add-plugin pyramid-orchestration
```

Or clone and point Cursor to the plugin directory.

### Codex

```bash
git clone https://github.com/JonnesLin/pyramid-orchestration.git ~/.codex/pyramid-orchestration

mkdir -p ~/.agents/skills
ln -s ~/.codex/pyramid-orchestration/skills/pyramid-orchestration ~/.agents/skills/pyramid-orchestration
```

Requires `multi_agent = true` in Codex config.

**Detailed docs:** [docs/README.codex.md](docs/README.codex.md)

### OpenCode

Add to your `opencode.json`:

```json
{
  "plugin": ["pyramid-orchestration@git+https://github.com/JonnesLin/pyramid-orchestration.git"]
}
```

### Gemini CLI

```bash
gemini extensions install https://github.com/JonnesLin/pyramid-orchestration
```

**Note:** Gemini CLI does not support subagent dispatch. The full automated pyramid is not available. See [GEMINI.md](GEMINI.md) for a manual session-based workflow that achieves the same context isolation.

### GitHub Copilot CLI

```bash
copilot plugin install pyramid-orchestration
```

### Verify Installation

Start a new session and try `/pyramid path/to/plan.md` or describe a task with 4+ subtasks. The agent should activate pyramid mode and dispatch Workers instead of handling everything directly.

## Usage

```
/pyramid path/to/plan.md
```

Or let the skill activate automatically when you're executing a plan with 4+ tasks.

## How It Works

1. **You** brainstorm and design with the main agent (interactive, stays in main context)
2. **Main** dispatches a Worker to produce the implementation plan
3. **Main** reads the plan, groups tasks, dispatches implementation Workers
4. **Workers** execute tasks using any available skills, dispatch their own subagents, commit code
5. **Workers** return one-line summaries to Main
6. **Main** dispatches review and verification Workers
7. **Main** reports results to you

The main agent never reads code, writes code, or runs tests. It coordinates.

## Platform Compatibility

| Platform | Subagent Support | Pyramid Mode | Notes |
|----------|-----------------|--------------|-------|
| Claude Code | Full | Full | Primary target |
| Cursor | Full | Full | Via plugin system |
| Codex | Via spawn_agent | Full | Requires multi_agent = true |
| OpenCode | Via subagent system | Full | Plugin-based install |
| Copilot CLI | Partial | Full | Via plugin marketplace |
| Gemini CLI | Not supported | Manual only | Use session-based workflow |

## Compatibility

Works with any skill ecosystem. Workers use whatever skills are available in the environment — TDD, code review, debugging, or just default agent capabilities.

The pyramid is an orchestration pattern, not a replacement for execution skills.

## Project Structure

```
pyramid-orchestration/
├── .claude-plugin/
│   └── plugin.json               # Claude Code plugin metadata
├── .cursor-plugin/
│   └── plugin.json               # Cursor plugin metadata
├── .codex/
│   └── INSTALL.md                # Codex installation guide
├── skills/
│   └── pyramid-orchestration/
│       └── SKILL.md              # Core orchestration logic
├── commands/
│   └── pyramid.md                # /pyramid slash command
├── agents/
│   └── worker.md                 # Worker Agent behavior template
├── docs/
│   └── README.codex.md           # Codex detailed docs
├── gemini-extension.json          # Gemini CLI extension manifest
├── GEMINI.md                      # Gemini CLI compatibility guide
├── README.md
└── LICENSE
```

## License

MIT
