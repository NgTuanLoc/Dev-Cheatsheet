---
tags: [claude-code, intermediate, slash-commands]
aliases: [Custom Commands, /command]
level: Intermediate
---

# Custom Slash Commands

> **One-liner**: A custom slash command is a markdown file with a prompt body — drop it in `.claude/commands/`, type `/<filename>`, and Claude runs it.

---

## Quick Reference

| Path | Scope | Visibility |
|------|-------|------------|
| `~/.claude/commands/<name>.md` | User-wide | All projects, only you |
| `<project>/.claude/commands/<name>.md` | Project-shared | Whoever clones the repo |
| (plugin)`<plugin>/commands/<name>.md` | Plugin-namespaced | `/<plugin>:<name>` |

| Frontmatter field | Purpose |
|-------------------|---------|
| `description` | One-liner for `/help` listing |
| `argument-hint` | Hint shown when typing the command |
| `model` | Override model for this command |
| `allowed-tools` | Restrict tool list for this run |

| In the body | What it does |
|-------------|--------------|
| Plain markdown | Becomes the prompt |
| `$ARGUMENTS` | Replaced with everything after `/cmd` |
| `$1`, `$2` | First / second positional arg |
| `!<command>` | Execute a shell command, splice output |

---

## Core Concept

A custom slash command is just a markdown file. Its filename (minus `.md`) becomes the command name. Type `/foo` and the body of `foo.md` is sent to Claude as a prompt — possibly with arguments substituted in.

Use them to capture **repeatable workflows**: "review my branch", "draft a release note", "scaffold a new endpoint matching the existing pattern". Anything you'd write twice is a candidate.

Project-shared commands live in `<project>/.claude/commands/` and travel with the repo — onboard new teammates with one `git clone`. User-wide commands in `~/.claude/commands/` are personal habits that follow you across projects.

The body is plain markdown. Treat it like writing a clear ticket — what the goal is, what's in scope, what "done" looks like. The better the prompt, the better the command.

---

## Syntax & API

### Minimal command

```markdown
---
description: Run the test suite and summarise failures
---

Run `npm test`. If anything fails, group failures by file and show
the first error per group. If everything passes, just say "✓".
```

> Saved as `.claude/commands/test-summary.md`. Invoke with `/test-summary`.

### Command with arguments

```markdown
---
description: Open a PR with the given title
argument-hint: <pr title>
---

Create a PR for the current branch with title:

  $ARGUMENTS

Body should include:
- 3-bullet summary of what changed
- test plan checklist with concrete TODOs
```

> Invoke: `/pr Add pagination to /users`. `$ARGUMENTS` becomes the rest of the line.

### Command that runs a shell hook in its prompt

```markdown
---
description: Review the current diff
---

Here is the diff vs main:

```
!git diff main...HEAD
```

Review for: correctness, security, missing tests. Be terse.
List up to 5 issues per category, ranked.
```

> The `!git diff main...HEAD` runs the command and splices its output into the prompt before Claude sees it. (Behaviour varies by version — check your docs.)

### Command with restricted tools

```markdown
---
description: Read-only repo audit
allowed-tools: [Read, Glob, Grep]
model: haiku
---

Audit `src/` for TODO and FIXME comments. List file:line and the comment.
Do not change anything.
```

### Plugin-namespaced command

```text
> /myplugin:scaffold-endpoint users
```

Used when a plugin contributes commands; the `<plugin>:` prefix prevents collisions.

---

## Common Patterns

### Pattern: scaffold-by-template

```markdown
---
description: Scaffold a new API route mirroring existing pattern
argument-hint: <route name>
---

Create a new route called `$ARGUMENTS`, mirroring `src/api/health.ts`:
- controller, service, test files
- registered in `src/api/index.ts`
- minimal handler returning 200

No business logic yet — just the skeleton.
```

### Pattern: pre-commit ritual

```markdown
---
description: Run pre-commit checks
---

In order, fail-fast:
1. `npm run lint` — fix auto-fixable, report rest
2. `npm run typecheck`
3. `npm test`

Report each step's result. Stop at the first failure and explain.
```

### Pattern: report builder

```markdown
---
description: Branch ship-readiness report
---

Audit the current branch for ship-readiness:
- uncommitted changes? (`git status`)
- ahead of main? (`git log main..HEAD`)
- tests for new code?
- CI green?

Output a punch list: done vs missing. Under 200 words.
```

### Pattern: shareable onboarding

```markdown
---
description: How to run this project locally
---

Walk me through running this project locally end-to-end:
- prerequisites (read README + package.json)
- env vars needed
- DB / external services
- exact commands

Confirm each step works before moving on.
```

---

## Gotchas & Tips

- **Project commands win for the same name** — `<project>/.claude/commands/` overrides `~/.claude/commands/` for that project.
- **Frontmatter fields vary by version.** Some installs accept `model`, `allowed-tools`, `argument-hint`; others only `description`. Read your docs.
- **The body is a prompt, not a script.** It's interpreted by Claude, not the shell. The `!` syntax (when supported) is the bridge to the shell.
- **Avoid one giant command** that tries to do everything. Smaller composable commands are easier to debug and reuse.
- **Document the args.** A user typing `/pr` without args has no idea what's expected — `argument-hint` shows up in the picker.
- **Commands inherit your permissions** — they don't elevate. A `/risky-thing` command will still prompt unless the rules allow it.
- **Custom commands are read fresh on each invocation** — edit the `.md` and re-invoke; no restart needed.
- **Don't name a command the same as a built-in** (`/help`, `/clear`). Built-ins win and your file is shadowed.

---

## See Also

- [[03 - Slash Commands Basics]]
- [[01 - Subagents]]
- [[04 - Hooks]]
- [[03 - settings.json]]
