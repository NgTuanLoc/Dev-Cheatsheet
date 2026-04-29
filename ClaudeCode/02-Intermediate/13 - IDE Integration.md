---
tags: [claude-code, intermediate, ide]
aliases: [VS Code, JetBrains, IDE, Editor Integration]
level: Intermediate
---

# IDE Integration

> **One-liner**: Claude Code is a CLI first, but its IDE extensions add inline context — current file, selection, and diagnostics flow into the prompt automatically.

---

## Quick Reference

| Surface | Best for |
|---------|----------|
| Terminal CLI | Power users, scripting, full control |
| VS Code extension | Inline edits, selection-aware prompts, diff review |
| JetBrains plugin | IntelliJ / Rider / PyCharm users — same idea, native UI |
| Desktop app | Multi-project, no terminal needed |
| Web (claude.ai/code) | Quick questions, no install |

| Feature | CLI | VS Code | JetBrains |
|---------|-----|---------|-----------|
| Selection auto-attached | manual paste | ✅ | ✅ |
| Diagnostics in prompt | manual paste | ✅ | ✅ |
| Diff preview before write | textual | ✅ visual | ✅ visual |
| Slash commands | ✅ | ✅ | ✅ |
| Hooks / settings | ✅ | ✅ | ✅ |
| Subagents | ✅ | ✅ | ✅ |

---

## Core Concept

Underneath, every surface runs the same agent against the same project. What differs is **how context flows in**:

- In the **terminal**, you paste or describe context yourself.
- In the **IDE**, the extension passes your current file, selection, cursor position, and diagnostics (errors, warnings) automatically. You don't have to say "the file at `src/api/users.ts` line 47" — the extension already told Claude.

That convenience is the main win. Everything else (settings, hooks, slash commands, subagents) is identical to the CLI.

A common workflow is **terminal-in-IDE**: open a terminal split inside VS Code/JetBrains, run `claude` there. You get IDE editing + CLI control, and the extension still wires up selection context.

---

## Syntax & API

### Install (VS Code)

```bash
# From the VS Code marketplace, search "Claude Code"
# Or via CLI:
code --install-extension anthropic.claude-code
```

After install: open the Claude panel (sidebar icon) or run **Claude: Start session** from the command palette.

### Install (JetBrains — IntelliJ, Rider, PyCharm, GoLand, WebStorm)

```text
Settings → Plugins → Marketplace → search "Claude Code" → Install → Restart IDE
```

### Common keyboard shortcuts (VS Code defaults)

| Action | Shortcut |
|--------|----------|
| Open Claude panel | `Ctrl/Cmd + Shift + L` |
| Send selection to Claude | `Ctrl/Cmd + L` |
| Apply suggested edit | `Ctrl/Cmd + Enter` |
| Reject suggested edit | `Esc` |

> Shortcuts are configurable. Run **Preferences: Open Keyboard Shortcuts** and search "Claude" to rebind.

### Selection-aware prompting

Highlight a function, then:

```text
> explain this. Focus on the side-effects.
```

The extension prepends the selection automatically. No paste needed.

### Diagnostics in scope

If your file has TypeScript errors, ESLint warnings, or compiler diagnostics:

```text
> fix the errors in this file
```

The extension forwards the diagnostics list. Claude addresses each one.

### Terminal-in-IDE pattern

```bash
# Open a terminal pane inside VS Code (Ctrl+`)
claude
> work on src/api/users.ts
```

You edit in the IDE, Claude runs in the terminal, both see the same files. Fast iteration loop.

---

## Common Patterns

### Pattern: refactor selection with constraints

Highlight a function:

```text
> Refactor this for early returns and pull the validation into a helper.
  Don't change the function signature. Don't add new dependencies.
```

The extension passes the selection; Claude returns a diff you can apply visually.

### Pattern: explain unfamiliar code

```text
> walk me through what this does, in 5 bullets.
  Where does it get called from?
```

Claude reads the selection, searches the project for callers, returns a summary.

### Pattern: scoped fix

```text
> the linter is unhappy with this file. Fix only the lint issues —
  no behaviour changes, no reorganisation.
```

### Pattern: pair-program loop

1. Write a failing test in the IDE.
2. Highlight it. "Make this pass — minimal change."
3. Apply the diff.
4. Refactor with another prompt.

Use [[10 - TDD Workflow]] for the discipline.

### Pattern: diff-review before applying

The extension shows proposed edits as a side-by-side diff. Read it before clicking Apply. Claude can hallucinate; the diff view is your last sanity check.

### Pattern: project-wide search-and-edit

Even in the IDE, large changes are easier in CLI mode (Plan mode → execute). Use the IDE for *spot* edits and the terminal for *broad* sweeps.

---

## Gotchas & Tips

- **The extension and CLI share `~/.claude` settings.** A hook configured for the CLI runs in the IDE too. A `settings.json` change applies to both.
- **The IDE doesn't auto-save.** If Claude edits a file, your IDE may show "modified externally" — accept the reload or risk overwriting.
- **Extension vs terminal-in-IDE**: the extension surfaces a panel; terminal-in-IDE just runs the CLI. Many power users prefer terminal-in-IDE for full control.
- **Selection context is convenient but lossy.** Claude only sees what's selected — if behaviour depends on imports above, include them or expand the selection.
- **Apply selectively.** When Claude proposes multi-file edits, the diff view lets you accept some files and skip others.
- **Different IDE, same agent.** Don't relearn — switching from VS Code to JetBrains is a UI change, not a Claude change.
- **Watch out for stale diagnostics.** If you applied an edit but the language server hasn't reindexed, follow-up prompts may see old errors. Save/reload first.
- **Web (claude.ai/code) is sandboxed differently** — no local file access in the same way; better for code-explanation Q&A than active editing.
- **The desktop app** is a wrapper — same engine, but easier multi-project switching than the CLI.
- **Hooks run in your local shell**, regardless of which surface launched the tool call. Test hooks from the CLI; they'll behave the same in the IDE.

---

## See Also

- [[02 - Installation and Setup]]
- [[03 - Slash Commands Basics]]
- [[14 - Multi-file Refactoring]]
- [[10 - TDD Workflow]]
