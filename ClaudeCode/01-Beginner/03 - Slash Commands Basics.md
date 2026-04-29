---
tags: [claude-code, beginner, slash-commands]
aliases: [Slash Commands, /commands]
level: Beginner
---

# Slash Commands Basics

> **One-liner**: Slash commands are reusable macros — type `/<name>` to invoke a built-in capability or a custom workflow you (or your team) wrote.

---

## Quick Reference — Built-in commands

| Command | What it does |
|---------|--------------|
| `/help` | List available commands |
| `/clear` | Clear the conversation, keep settings |
| `/init` | Generate a starter `CLAUDE.md` for the project |
| `/memory` | Open the memory file for editing |
| `/agents` | List or run subagents |
| `/review` | Review the current PR or diff |
| `/config` | Open settings UI / edit `settings.json` |
| `/cost` | Show session cost so far |
| `/model` | Switch model mid-session |
| `/permissions` | Show / edit permission rules |
| `/mcp` | List configured MCP servers |
| `/login` / `/logout` | Auth |
| `/exit` (or `Ctrl+D`) | End session |

> Available commands depend on your version. Run `/help` to see what's actually installed.

---

## Core Concept

A slash command is just a chunk of pre-written prompt that you can invoke by name. It can be **built-in** (shipped with Claude Code) or **custom** (a `.md` file you wrote in `~/.claude/commands/` or `<project>/.claude/commands/`).

Use slash commands when you find yourself typing the same prompt twice. The cost is tiny — one markdown file with a description and a body — and the payoff is a tight, reusable workflow your whole team can share.

Custom slash commands can take **arguments** (`/loop 5m /foo`), reference **frontmatter** for metadata, and be **scoped** (user-level vs project-level).

---

## Syntax & API

### Invoke a built-in

```text
> /help
> /init
> /review 1234           # review PR #1234
> /clear
```

### Inspect what a custom command does

```bash
# At the shell — read the markdown file directly
cat ~/.claude/commands/my-command.md
cat .claude/commands/team-command.md
```

### Switch model mid-session

```text
> /model
# pick from the list, or:
> /model claude-haiku-4-5
```

### Check session cost

```text
> /cost
```

---

## Common Patterns

### Workflow: review-then-fix

```text
> /review
# Claude posts a review
> ok, address the CRITICAL items
```

### Workflow: clear context between unrelated tasks

```text
> /clear
# fresh slate, settings preserved, CLAUDE.md still loaded
```

### Workflow: discoverability for new teammates

```text
> /help          # see built-ins
> /agents        # see project + user subagents
> /mcp           # see configured MCP servers
```

---

## Gotchas & Tips

- **Slash commands are *prompt macros*, not shell commands.** They expand into a prompt that the model executes — they don't bypass model reasoning.
- **A custom command's quality depends on its prompt.** A vague body = vague behavior. Spell out the steps you want.
- **Project commands beat user commands** for the same name — local config wins.
- **Some commands take arguments after the name.** `/loop 5m /check-build` runs `/check-build` every 5 minutes. Read the command's body to learn its arg shape.
- **`/clear` does NOT log out** — it just resets the conversation. Use `/logout` to clear auth.
- **`/init` is a one-shot.** Re-running it overwrites your `CLAUDE.md`. Edit by hand instead, or commit your changes before regenerating.
- **`/cost` reflects the current session only.** Total bill lives in your Anthropic account dashboard.

---

## See Also

- [[02 - Custom Slash Commands]] — write your own
- [[06 - CLAUDE.md Files]]
- [[01 - Subagents]]
- [[05 - MCP Servers Using]]
