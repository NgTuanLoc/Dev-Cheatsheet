---
tags: [claude-code, intermediate, settings]
aliases: [settings.json, Settings, Configuration]
level: Intermediate
---

# settings.json

> **One-liner**: `settings.json` is where you tell Claude Code your model, permissions, env vars, hooks, and MCP servers — three layers (user, project, project-local) that cascade.

---

## Quick Reference

### File locations and precedence

| Level | Path | Commit? | Wins over |
|-------|------|---------|-----------|
| Enterprise | OS-managed (varies) | (managed) | everything below |
| User | `~/.claude/settings.json` | no | project (for user-only fields) |
| Project shared | `<project>/.claude/settings.json` | yes | user (for project-relevant fields) |
| Project local | `<project>/.claude/settings.local.json` | **no** | project shared |

### Top-level keys
| Key | Purpose |
|-----|---------|
| `model` | Default model (e.g. `claude-sonnet-4-6`) |
| `permissions` | `allow` / `deny` rules + `defaultMode` |
| `env` | Environment variables Claude inherits |
| `hooks` | Hook commands by event |
| `mcpServers` | MCP server registrations |
| `agents` | Agent overrides (rare) |
| `output` | Default verbosity / styling |
| `cleanupPeriodDays` | Transcript retention |

---

## Core Concept

`settings.json` is the configuration spine. Keep what *everyone* on the team needs in `<project>/.claude/settings.json` (commit it). Keep what *only you* need in `~/.claude/settings.json` or `.claude/settings.local.json`. The cascade resolves at startup; arrays merge, scalars override.

Most people only edit two things:
1. `permissions.allow` — to stop being prompted for safe commands
2. `model` — to pin Sonnet (or escalate to Opus / drop to Haiku)

Hooks, MCP servers, and env vars come later as you need them.

---

## Syntax & API

### Minimal user-level settings

```json
{
  "model": "claude-sonnet-4-6",
  "permissions": {
    "defaultMode": "default",
    "allow": [
      "Read",
      "Glob",
      "Grep",
      "Bash(git status)",
      "Bash(git diff:*)",
      "Bash(git log:*)"
    ],
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(git push --force:*)"
    ]
  }
}
```

### Project-shared settings

```json
{
  "permissions": {
    "allow": [
      "Bash(npm test:*)",
      "Bash(npm run lint:*)",
      "Bash(npm run build)",
      "Bash(npx tsc:*)"
    ]
  },
  "env": {
    "NODE_ENV": "development"
  }
}
```

> Commit this. Teammates inherit safe project commands without each writing their own.

### Project-local override

```json
{
  "permissions": {
    "allow": [
      "Bash(./scripts/local-only-thing:*)"
    ]
  },
  "env": {
    "LOCAL_DB_URL": "postgres://localhost/dev"
  }
}
```

> Add to `.gitignore`. This is yours, not the team's.

### Hooks

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "npx prettier --write ${file}"
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "./scripts/audit-bash.sh"
      }
    ]
  }
}
```

> See [[04 - Hooks]] for the full hook contract.

### MCP servers

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/repos"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${env:GH_TOKEN}" }
    }
  }
}
```

> See [[05 - MCP Servers Using]] for adding servers.

### Output / verbosity

```json
{
  "output": {
    "style": "terse",
    "showThinking": false
  }
}
```

---

## Common Patterns

### Pattern: per-language test allowlist

```json
{
  "permissions": {
    "allow": [
      "Bash(pytest:*)",
      "Bash(python -m pytest:*)",
      "Bash(coverage run:*)",
      "Bash(coverage report:*)"
    ]
  }
}
```

### Pattern: read-only audit settings

```json
{
  "permissions": {
    "defaultMode": "plan",
    "allow": ["Read", "Glob", "Grep", "WebSearch", "WebFetch"]
  }
}
```

> Use for security audits or third-party code reviews — Claude can read everything but can't change anything.

### Pattern: model per project

A monorepo with a perf-critical service might pin Opus; a small CLI utility might pin Haiku.

```json
// services/billing/.claude/settings.json
{ "model": "claude-opus-4-7" }
```

```json
// scripts/.claude/settings.json
{ "model": "claude-haiku-4-5" }
```

### Pattern: alias env vars from your shell

```json
{
  "env": {
    "DATABASE_URL": "${env:DATABASE_URL}",
    "OPENAI_API_KEY": "${env:OPENAI_API_KEY}"
  }
}
```

> Don't paste secrets here — reference them from your shell environment.

---

## Gotchas & Tips

- **Local settings shadow shared.** A teammate adding to `.local.json` will not see their changes affect anyone else; that's the point.
- **Arrays merge, scalars override.** `permissions.allow` from user + project both apply; `model` is overridden by the highest-precedence level that sets it.
- **Don't commit `.local.json`.** Add to `.gitignore` once: `**/.claude/settings.local.json`.
- **Validate JSON.** A trailing comma silently disables your settings on some parsers — use a JSON-aware editor.
- **Pattern syntax for permissions** is `Tool` (any), `Tool(exact)`, or `Tool(prefix:*)`. Get this wrong and the rule never matches.
- **`bypassPermissions` here is the same as the CLI flag** — danger compounds. Don't.
- **Restart not required for most changes** — settings re-read at session start, sometimes mid-session via `/config`. When in doubt, restart.
- **Enterprise settings can lock fields** — if a setting "won't take," check whether your org has imposed an enterprise policy.

---

## See Also

- [[02 - Installation and Setup]]
- [[05 - Permissions and Safety]]
- [[04 - Hooks]]
- [[05 - MCP Servers Using]]
