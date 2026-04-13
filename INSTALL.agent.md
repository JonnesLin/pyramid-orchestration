# AI Installation Instructions for Pyramid Orchestration

This file is the unified installation and update entrypoint for AI agents.

## Goal

Install or update `pyramid-orchestration` for the current platform, using the
platform-native layout when possible.

## General Rules

1. Detect the current platform first. Prefer runtime/tooling context over
   filesystem guessing.
2. Prefer updating an existing installation over creating a second copy.
3. Prefer a full-repo install when the platform supports plugins or repo-based
   extensions. This project includes skills, commands, agents, and plugin
   metadata.
4. Only use a skill-only install as a fallback when the platform cannot use the
   full repo layout.
5. Do not delete unrelated files or directories. If you need to fix an
   existing symlink, replace only that symlink.
6. After finishing, report:
   - installed repo path
   - active branch
   - current commit hash
   - activation path used by the platform
   - whether the user needs to restart the tool

## Repository

- Repo URL: `https://github.com/JonnesLin/pyramid-orchestration.git`
- Default branch: `main`

## Platform Detection Order

Follow the first branch that clearly matches the current environment.

### Codex

Use this branch if you are running inside Codex or the environment clearly uses
Codex conventions.

1. Install or update the repo at `~/.codex/pyramid-orchestration`
2. Ensure the skill activation path exists:
   `~/.agents/skills/pyramid-orchestration ->
   ~/.codex/pyramid-orchestration/skills/pyramid-orchestration`
3. For Codex-specific details, follow the instructions in:
   `.codex/INSTALL.md`
4. Tell the user to restart Codex after installation or update

### Claude Code

Use this branch if you are running inside Claude Code.

Preferred approach:

1. Install or update the repo at
   `~/.claude/plugins/repos/pyramid-orchestration`
2. Keep the full repo layout so Claude Code can use skills, commands, agents,
   and plugin metadata together

Fallback approach if the current Claude Code setup only supports local skills:

1. Install or update the repo in a temporary or durable local path
2. Copy or link `skills/pyramid-orchestration` into
   `~/.claude/skills/pyramid-orchestration`

Always tell the user to restart Claude Code after installation or update.

### Gemini CLI

Use this branch if you are running inside Gemini CLI.

1. Run:
   `gemini extensions install https://github.com/JonnesLin/pyramid-orchestration`
2. If the extension is already installed, update it using Gemini's extension
   update flow if available
3. If Gemini cannot automate the full install, use the manual workflow in
   `GEMINI.md`

### Cursor

Use this branch if you are running inside Cursor.

Preferred approach:

1. Use Cursor's native plugin flow if it is available in the current runtime
2. Install or update this repo as the plugin source

Fallback approach:

1. Clone or update the repo in a local path
2. Point Cursor at the repo's plugin directory if manual registration is still
   required

### OpenCode

Use this branch if you are running inside OpenCode.

1. Ensure the user's `opencode.json` includes:
   `"pyramid-orchestration@git+https://github.com/JonnesLin/pyramid-orchestration.git"`
2. If it is already configured, update the underlying repo or refresh the
   plugin according to OpenCode's normal flow

### GitHub Copilot CLI

Use this branch if you are running inside GitHub Copilot CLI.

1. Run:
   `copilot plugin install pyramid-orchestration`
2. If already installed, update it using Copilot CLI's normal plugin update
   path if available

### Unknown or Mixed Environment

If the environment is unclear:

1. Say which signals were ambiguous
2. Prefer a conservative full-repo clone to a neutral path such as
   `~/pyramid-orchestration`
3. Do not pretend activation is complete unless you can verify the platform's
   activation path

## Update Policy

When you find an existing repo clone at the platform's expected path:

1. Enter that repo
2. Run `git pull --ff-only`
3. Repair activation links only if they are missing or incorrect

## Verification

Before claiming success, verify:

1. The repo exists at the expected path
2. `git branch --show-current` returns the active branch
3. `git rev-parse --short HEAD` returns a commit hash
4. The platform activation path exists and points to the expected skill or repo

## Final Report

Report the result in this format:

- Platform detected: `...`
- Repo path: `...`
- Branch: `...`
- Commit: `...`
- Activation path: `...`
- Restart required: `yes` or `no`
