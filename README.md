# Pyramid Orchestration

File-backed pyramid orchestration for AI coding agents.

## The Problem

AI agents using flat delegation hit context limits fast. One main agent
dispatching 8+ jobs accumulates code, reviews, and test output in a single
context window. The result: frequent compaction, information loss, degraded
performance.

Many runtimes also support only **top-level** agent dispatch. The main session
can create agents, but those agents cannot reliably create more agents of their
own. A pyramid protocol that depends on true nested delegation is therefore
brittle across tools.

## The Solution

Keep the pyramid mental model, but make handoffs durable with files:

```
Main Agent (Orchestrator)
  ├── summary ← Worker A
  │              ├── request file → Child A-01
  │              └── continuation Worker A absorbs result file
  ├── summary ← Worker B
  │              ├── request file → Child B-01
  │              └── continuation Worker B absorbs result file
  └── summary ← Worker C
```

- **Workers → request files**: delegated child work is described in durable
  mailbox files
- **Main → child agents**: Main is the only dispatcher
- **Children → result files**: detailed output stays on disk
- **Workers → Main**: summaries only
- **Workers are disposable**: their context is released after each handoff or
  completion

The main agent's context stays light. Detailed work lives in files and git
commits.

## Context Savings

| Mode | 8 jobs | Main context |
|------|--------|--------------|
| Flat | All results to Main | ~29,000 tokens |
| Pyramid (3 Workers) | Summaries plus request ids | ~5,600 tokens |

~5x reduction. Workers handle detailed reasoning, then release.

## When to Use

| Delegated work needed | Recommendation |
|----------------------|----------------|
| 1-3 | Flat delegation |
| 4-8 | Pyramid with 2-3 Workers |
| 9+ | Pyramid with 3-5+ Workers |

## AI Installation

For AI agents, use this instruction:

```text
Fetch and follow instructions from https://raw.githubusercontent.com/JonnesLin/pyramid-orchestration/refs/heads/main/INSTALL.agent.md
```

That unified entrypoint tells the AI to detect the current platform and then
install or update the project using the correct platform-specific flow.

## Manual Installation by Platform

**Note:** Pyramid Orchestration needs the **main session** to be able to create
independent agents. It does not require Worker-created agents. Platforms
without any agent dispatch can still use the manual workflow described in the
Gemini CLI section.

### Claude Code

Clone directly into the plugins repos directory:

```bash
git clone https://github.com/JonnesLin/pyramid-orchestration.git ~/.claude/plugins/repos/pyramid-orchestration
```

Or install just the skill:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/JonnesLin/pyramid-orchestration.git /tmp/pyramid-orchestration
cp -r /tmp/pyramid-orchestration/skills/pyramid-orchestration ~/.claude/skills/pyramid-orchestration
rm -rf /tmp/pyramid-orchestration
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

Requires main-session agent dispatch in Codex. In many setups that means
`multi_agent = true`.

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

**Note:** Gemini CLI does not support agent dispatch. The automated pyramid is
not available. See [GEMINI.md](GEMINI.md) for a manual session-based workflow
that uses the same file-backed handoff pattern.

### GitHub Copilot CLI

```bash
copilot plugin install pyramid-orchestration
```

### Verify Installation

Start a new session and try `/pyramid path/to/plan.md` or describe a task with
4+ subtasks. The agent should activate pyramid mode, create a run directory
under `.pyramid/runs/`, dispatch Workers, and keep detailed results in files
rather than the main chat.

## Usage

```
/pyramid path/to/plan.md
```

Or let the skill activate automatically when you're executing a plan with 4+
tasks.

## How It Works

1. **You** brainstorm and design with the main agent
2. **Main** dispatches a Worker to produce the implementation plan
3. **Main** reads the plan, groups tasks, creates a run directory, and
   dispatches implementation Workers
4. **Workers** execute what they can directly, write request files when they
   need deeper delegated work, and commit code
5. **Main** fulfills request files by dispatching child agents and routing
   result files back into continuation Workers
6. **Workers** return one-line summaries to Main
7. **Main** dispatches review and verification Workers using the same
   file-backed protocol
8. **Main** reports results to you

The main agent never reads code, writes code, or runs tests. It coordinates.

## Platform Compatibility

| Capability | Example platforms | Pyramid mode | Notes |
|------------|-------------------|--------------|-------|
| Main session can dispatch agents | Claude Code, Codex, compatible plugin hosts | Automated | Main dispatches both Workers and child agents; Workers use mailbox files |
| No agent dispatch available | Gemini CLI | Manual only | Use separate sessions with the same `.pyramid/runs/<run-id>/` structure |

## Compatibility

Works with any skill ecosystem. Workers use whatever skills are available in
the environment — TDD, code review, debugging, or just default agent
capabilities.

The pyramid is an orchestration pattern, not a replacement for execution
skills.

## Project Structure

```
pyramid-orchestration/
├── .claude-plugin/
│   └── plugin.json               # Claude Code plugin metadata
├── .cursor-plugin/
│   └── plugin.json               # Cursor plugin metadata
├── INSTALL.agent.md              # Unified AI installation entrypoint
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
├── gemini-extension.json         # Gemini CLI extension manifest
├── GEMINI.md                     # Gemini CLI compatibility guide
├── README.md
└── LICENSE
```

## License

MIT
