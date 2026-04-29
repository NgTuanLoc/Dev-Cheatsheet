---
tags: [claude-code, beginner, cli, settings]
aliases: [Install, Setup, First Run]
level: Beginner
---

# Installation and Setup

> **One-liner**: Install via npm (or your package manager), log in once, drop a `CLAUDE.md` in your project, and you're running.

---

## Quick Reference

| Step | Command |
|------|---------|
| Install (npm) | `npm install -g @anthropic-ai/claude-code` |
| Verify | `claude --version` |
| Launch | `claude` (in your project root) |
| Login | First run prompts for browser login |
| Logout | `claude logout` |
| Update | re-run install command |
| See help | `claude --help` or `/help` inside a session |

| Path | What lives there |
|------|------------------|
| `~/.claude/settings.json` | User-wide settings (allowedTools, env, hooks) |
| `~/.claude/agents/` | User-wide subagent definitions |
| `~/.claude/commands/` | User-wide custom slash commands |
| `~/.claude/projects/` | Per-project state, transcripts, memory |
| `<project>/.claude/settings.json` | Project-shared settings (commit this) |
| `<project>/.claude/settings.local.json` | Project-local overrides (gitignore this) |
| `<project>/.claude/agents/` | Project-shared subagents |
| `<project>/.claude/commands/` | Project-shared slash commands |
| `<project>/CLAUDE.md` | Auto-loaded project context |

---

## Core Concept

Installation is a one-time global install plus a one-time login. After that, `claude` launches a session in whatever directory you're standing in, picks up the project's `CLAUDE.md` and `.claude/` config, and starts the agent loop.

Settings cascade: **enterprise → user → project (shared) → project (local)**. Lower-precedence files set defaults; higher-precedence files override. Permissions stack (e.g., a tool allowed at the user level is also allowed at the project level).

Once installed, you'll typically configure two things on day one: (1) a `CLAUDE.md` per project so Claude knows the codebase, and (2) an `allowedTools` list in your settings so you stop being prompted for safe operations.

---

## Syntax & API

### Install

```bash
# npm — works everywhere Node is installed
npm install -g @anthropic-ai/claude-code

# Verify
claude --version
claude --help
```

### First run

```bash
cd ~/code/your-project
claude

# It opens a browser to log in to your Anthropic account.
# Once authed, the prompt is ready.
```

### Generate a starter CLAUDE.md

```bash
# Inside a session, in your project root
> /init
```

`/init` reads the repo and writes a `CLAUDE.md` summarising languages, scripts, and conventions. Edit it after — `/init` is a starting point, not the final answer.

### Pick a model at launch

```bash
claude --model claude-sonnet-4-6
claude --model claude-opus-4-7
claude --model claude-haiku-4-5
```

### Headless / one-shot

```bash
claude -p "list TODO comments in src/" --output-format json
```

### Update / uninstall

```bash
npm install -g @anthropic-ai/claude-code   # re-install to update
npm uninstall -g @anthropic-ai/claude-code
```

---

## Common Patterns

### Sample `~/.claude/settings.json` to start with

```json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "allow": [
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(git log:*)",
      "Read",
      "Glob",
      "Grep"
    ]
  }
}
```

> Add commands you trust. See [[03 - settings.json]] for the full schema and [[05 - Permissions and Safety]] for safe defaults.

### Sample project `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test:*)",
      "Bash(npm run lint:*)",
      "Bash(npm run build)"
    ]
  }
}
```

> Commit this. Teammates inherit it. Local-only overrides go in `.claude/settings.local.json` (gitignore that one).

---

## Gotchas & Tips

- **`/init` is not perfect** — it generates a starter `CLAUDE.md`; treat it as a draft and edit.
- **Permission prompts feel noisy at first.** The fix is allowlisting safe commands in `settings.json`, not bypassing all prompts. See [[05 - Permissions and Safety]].
- **Don't commit `.claude/settings.local.json`** — it's local-only by design. Add it to `.gitignore`.
- **One project per `claude` session** — launch from the project root so paths and `CLAUDE.md` resolve correctly.
- **Updates are not automatic.** Re-run the install command periodically (or use a wrapper that does).
- **Multiple terminals = multiple sessions** — each gets its own context. Long sessions accumulate context; restart with `/clear` or by exiting.
- **First run downloads nothing weird** — auth is browser-based and stored in `~/.claude/`. To switch accounts: `claude logout` then re-launch.

---

## See Also

- [[03 - Slash Commands Basics]]
- [[03 - settings.json]]
- [[05 - Permissions and Safety]]
- [[06 - CLAUDE.md Files]]
