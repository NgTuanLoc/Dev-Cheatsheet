---
tags: [claude-code, intermediate, settings]
aliases: [Output Style, Verbosity, Persona]
level: Intermediate
---

# Output Styles

> **One-liner**: Tune how chatty Claude is — terse for quick fixes, verbose for teaching, or a custom persona for a project's voice — without rewriting prompts every time.

---

## Quick Reference

| Style | Behaviour |
|-------|-----------|
| `default` | Concise, code-first, brief end-of-turn summary |
| `terse` | Minimum prose, near-zero prologue, no trailing summary |
| `verbose` | Walks through reasoning, explains tradeoffs |
| `teach` / `explanatory` | Pauses to teach concepts as it goes |
| custom | A markdown file you author |

| Setting | Where |
|---------|-------|
| `output.style` | `settings.json` |
| `/output` (or `/style`) | Switch mid-session |
| `~/.claude/output-styles/<name>.md` | Custom user style |
| `<project>/.claude/output-styles/<name>.md` | Custom project style |

---

## Core Concept

Output styles control **how** Claude communicates, not **what** it does. The work is the same; the framing changes. A `terse` reviewer says "Lines 12-18 leak the DB connection. Fix: wrap in try/finally." A `verbose` reviewer adds three paragraphs about why.

Pick by task:
- **Terse** for code review, automation, CI integration
- **Default** for daily work
- **Verbose / teach** when learning a new codebase or onboarding
- **Custom** when your team wants a consistent voice

A custom style is a markdown file with instructions about voice, structure, and formatting — Claude composes it into its system prompt at session start.

---

## Syntax & API

### Pin a style globally

```json
{
  "output": {
    "style": "terse"
  }
}
```

### Switch mid-session

```text
> /output verbose
```

### Custom style file

```markdown
---
name: senior-reviewer
description: Voice of a senior engineer in code review
---

You are speaking as a senior engineer reviewing a junior's PR. Be:
- direct, no hedging
- specific about file:line
- focused on correctness, security, and tests
- silent on style nits

When you find an issue, output:
- severity (CRITICAL / HIGH / MEDIUM / LOW)
- file:line
- one-line description
- minimal fix sketch

Do not write trailing summaries.
```

> Save as `<project>/.claude/output-styles/senior-reviewer.md`. Switch with `/output senior-reviewer` or pin in `settings.json`.

### Compose styles per-command

```markdown
---
description: Run a senior-reviewer review of the current diff
output-style: senior-reviewer
---

Review the diff vs main. Use the senior-reviewer voice.
```

> A slash command can pin its own style for the duration of that command.

---

## Common Patterns

### Pattern: terse for CI, verbose for learning

User-level (`~/.claude/settings.json`):
```json
{ "output": { "style": "default" } }
```

Project-level for a CI repo (`<project>/.claude/settings.json`):
```json
{ "output": { "style": "terse" } }
```

### Pattern: persona that enforces project conventions

```markdown
---
name: drizzle-only
description: Reminds about ORM choice
---

Behave as a normal coding assistant, but: this codebase uses Drizzle ORM
exclusively. Never suggest Prisma, TypeORM, Sequelize, or raw SQL.
If asked, redirect to Drizzle equivalents.
```

> Effectively a "soft hook" that biases Claude's suggestions without writing code.

### Pattern: teach mode for onboarding

```text
> /output teach
> Walk me through how the auth pipeline works in this repo. Start
  from `app.use` and trace through.
```

In teach mode Claude pauses to explain mechanics that default mode would assume.

### Pattern: minimal for piped output

```bash
claude -p "list TODOs" --output-style terse > todos.txt
```

---

## Gotchas & Tips

- **Output style ≠ system prompt.** It customises voice and structure; it doesn't change tools, permissions, or capabilities.
- **Don't recreate `CLAUDE.md` rules in a style.** Project rules go in `CLAUDE.md`; persona/voice goes in the style. Mixing them is messy.
- **Project styles win for the same name.** Your `senior-reviewer.md` in the project shadows the user-level one.
- **A custom style is a prompt fragment** — write it tightly. Long, vague styles dilute behavior.
- **Switching mid-task can confuse Claude** — it may try to retroactively reformat. Switch at task boundaries.
- **`terse` does NOT mean "skip verification"** — Claude still runs tests, just reports outcomes briefly.
- **Custom styles don't automatically appear in `/output`** until the file exists with valid frontmatter.
- **Style is per-session.** New sessions read the configured default again.

---

## See Also

- [[03 - settings.json]]
- [[07 - Effective Prompting]]
- [[02 - Custom Slash Commands]]
- [[10 - Tips and Pitfalls]]
