---
tags: [claude-code, beginner, prompting]
aliases: [Tips, Pitfalls, Antipatterns]
level: Beginner
---

# Tips and Pitfalls

> **One-liner**: Most failed Claude sessions trace back to one of a handful of antipatterns — over-broad prompts, ignored permission prompts, and not verifying before declaring done.

---

## Quick Reference — Top antipatterns

| Antipattern | Symptom | Fix |
|-------------|---------|-----|
| "Fix everything" | Random changes everywhere | Bound scope to specific files |
| Pasting whole files | Wasted context, slow turns | Let Claude `Read` instead |
| Long single session | Drift, weird answers | `/clear` between unrelated tasks |
| Approving permissions blindly | Surprise commands run | Read the prompt; allowlist instead |
| "Looks fine to me" without tests | Silent regressions | Run tests, see output |
| Trusting "I have updated the file" | File may not be saved | Confirm with `git diff` or Read |
| Mixing TDD + refactor + feature | Sloppy result | One concern per prompt |
| Re-`/init` over edits | Loses your work | Edit by hand or commit first |

---

## Core Concept

Claude is a strong collaborator, not an oracle. Treat it like a fast junior engineer: clear scope, real verification, brief feedback. Most "bad sessions" come from skipping verification ("just trust the diff") or scope creep ("while you're in there, also...").

The two best habits:
1. **Verify after every meaningful change.** Run tests, look at diffs, read what changed. The cost is small; the cost of *not* doing it is regression chains that take days to unwind.
2. **`/clear` aggressively.** Long sessions drift. A 30-turn conversation about three different topics is worse than three 10-turn focused conversations.

---

## Syntax & API

### Verify after edits

```bash
# Built-in muscle memory
git status
git diff
npm test
npm run typecheck
```

### Reset context cleanly

```text
> /clear        # nukes conversation, keeps settings + CLAUDE.md
```

### Detect a session that's gone sideways

Symptoms:
- Claude keeps asking about things it already knows
- Edits are touching files unrelated to the task
- Tests fail in places you didn't change
- Cost is climbing without progress

Fix: stop, summarise where you are, `/clear`, restart with the summary as the prompt.

---

## Common Patterns

### Pattern: brief, frequent updates beat one mega-prompt

```text
# bad
> implement the entire user dashboard

# good
> step 1: scaffold src/dashboard/ with index, route, test stub.
  no logic yet. show me the file tree when done.
```

### Pattern: state your verification budget

```text
> Make the change. Run the tests once. If they pass, stop and report.
  If they fail, do NOT try to fix — show me the failure.
```

### Pattern: anchor in code, not in conversation

```text
# bad — Claude has to remember 20 turns ago
> like we discussed, do that thing for the auth module

# good — anchor explicitly
> like we discussed: extract `verifyJwt` from `auth.service.ts:42`
  into its own module `src/auth/verify-jwt.ts`. Tests in same shape.
```

### Pattern: explicit "stop here"

```text
> propose the diff but DO NOT apply it. paste the diff in your reply.
```

### Pattern: pre-flight check before risky ops

```text
> Before any deploy:
  1) git status — must be clean
  2) git pull --rebase — must be up-to-date
  3) npm test — must pass
  Stop and report at the first failure.
```

---

## Gotchas & Tips

### About context
- **Pasted output > 100 lines is usually wrong.** Save to a file and have Claude `Read` it.
- **Tool result text is invisible to you** unless Claude surfaces it. If you need to see something, ask.
- **Long sessions get expensive linearly** — Claude re-reads context every turn. Compress with `/clear` or summarisation.

### About edits
- **"I have updated the file" is not proof.** Run `git diff` or `Read` the file. (Claude is not lying; bugs in tools or permissions can cause silent failures.)
- **Edit + commit in the same turn** is risky if you can't see the diff. Stage, then ask for confirmation, then commit.
- **Don't let Claude rewrite tests to make them pass.** Tests are the spec; if they're wrong, fix them deliberately.

### About permissions
- **Read every Bash prompt.** Approving "destroy production" because you're tired is a real failure mode.
- **`bypassPermissions` is not a productivity hack.** Use a sandbox if you need to run wide.

### About model choice
- **Default to Sonnet.** It's the best coding model and the right balance for most work. Drop to Haiku for cheap automation; escalate to Opus for hard architectural problems. See [[15 - Model and Cost Optimization]].

### About when to stop
- **If you've reverted the same file three times, stop.** The prompt is wrong, not the implementation.
- **If you're correcting Claude on the same point twice, save it as a memory** — see [[06 - Memory System]].
- **If a session feels off, it is.** `/clear` and restart. The cost of restarting is one prompt; the cost of continuing is a bad outcome.

---

## See Also

- [[07 - Effective Prompting]]
- [[09 - Common Workflows]]
- [[05 - Permissions and Safety]]
- [[06 - Memory System]]
- [[15 - Model and Cost Optimization]]
