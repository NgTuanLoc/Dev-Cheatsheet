---
tags: [claude-code, intermediate, workflow]
aliases: [Code Review, /review, PR Review]
level: Intermediate
---

# Code Review with Claude

> **One-liner**: `/review` is a one-shot review of the current diff or PR; for serious reviews, use a fresh `code-reviewer` subagent so it judges the code without inheriting your reasoning.

---

## Quick Reference

| Approach | Command | Best for |
|----------|---------|----------|
| Built-in | `/review` | Quick sanity check on local diff |
| Built-in (PR#) | `/review 1234` | Review a GitHub PR |
| Subagent | "use the code-reviewer agent" | Independent, role-specialised review |
| Multi-agent | parallel forks with different lenses | High-stakes review |
| `/ultrareview` | (cloud, billed) | Expensive deep review across multiple models |

| Severity | Meaning | Action |
|----------|---------|--------|
| CRITICAL | Security or data-loss risk | **BLOCK** merge |
| HIGH | Correctness bug | **WARN** before merge |
| MEDIUM | Maintainability concern | Consider fixing |
| LOW | Style / nit | Optional |

---

## Core Concept

The simplest review is `/review`: Claude reads the diff and reports issues. It works, but it inherits whatever reasoning you've shared in the session. For an unbiased review, run a **fresh subagent** (`code-reviewer`) with no prior context.

For high-stakes changes, run **multiple agents in parallel** with different lenses (security, correctness, consistency, redundancy) and aggregate. See [[03 - Multi-Agent Reviews]].

A good review report:
- Groups issues by severity
- Anchors each to file:line
- Suggests a concrete fix sketch (not "consider refactoring")
- Skips style nits if a linter exists

---

## Syntax & API

### Built-in review

```text
> /review
# Reviews `git diff main...HEAD` (or current PR if on a PR branch)
```

```text
> /review 1234
# Reviews GitHub PR #1234 — uses gh CLI
```

### Subagent review (preferred for serious work)

```text
> have the code-reviewer agent review the diff vs main.
  Don't share my reasoning — I want fresh eyes.
```

> Requires a `code-reviewer` agent definition in `.claude/agents/`. See [[01 - Building Custom Agents]].

### Custom slash command for "review my branch"

```markdown
---
description: Independent review of current branch
---

Spawn a fresh `code-reviewer` subagent to review:

```
!git diff main...HEAD
```

Output: issues grouped by severity (CRITICAL / HIGH / MEDIUM / LOW),
each with file:line and a one-line fix sketch. Skip style nits.
```

> Save as `.claude/commands/review-branch.md`. Invoke with `/review-branch`.

### Security-focused review

```text
> have the security-reviewer agent audit the diff. Focus on:
  OWASP top 10, secret leakage, auth bypass, injection. Skip style.
```

---

## Common Patterns

### Pattern: review-then-fix loop

```text
> /review
# Claude posts a list

> address the CRITICAL items only. Leave the rest as TODOs.
```

### Pattern: scoped review

```text
> review the diff, but only files under `src/api/`.
  Issues at HIGH+ only. Under 200 words.
```

### Pattern: pre-merge gate

```bash
# In CI or as a local pre-push hook
claude -p "/review" --output-format json > review.json
# fail the build if any CRITICAL issue is reported
jq -e '.issues[] | select(.severity == "CRITICAL")' review.json && exit 1
```

### Pattern: triage the reviewer

```text
> The reviewer left these comments:
  <paste comments>

  Group by:
  - must-fix (correctness)
  - should-fix (maintainability)
  - nit (defer / decline)

  For each must-fix, propose the change.
```

### Pattern: review own PR via fork (no main-thread bias)

```text
> Fork an agent to review my PR. Tell it: this is a fresh review, do NOT
  read this conversation. Diff is `git diff main...HEAD`. Focus on
  correctness, security, missing tests. Be terse.
```

---

## Gotchas & Tips

- **Don't review with the same session that wrote the code.** It's biased toward "looks fine." Fork or fresh subagent.
- **`/review` reads diff against the configured base** — usually `main`. If you branched from somewhere else, set the base or pass it explicitly.
- **Reviews on auto-generated code waste tokens.** Skip schema/codegen output — tell Claude the path globs to ignore.
- **A clean review is suspicious.** If a 500-line diff has zero issues, ask again with a different lens (security / consistency).
- **Don't blindly apply review fixes.** Claude can hallucinate "fixes" for non-issues. Read each, then apply.
- **`/ultrareview` is user-triggered and billed** — Claude can't auto-launch it. Suggest it; user confirms.
- **Security reviews need the OWASP frame** — a generic review misses auth-bypass and injection patterns. Use `security-reviewer` explicitly.
- **Tests are part of the review.** "Are there tests for the new behaviour?" — if not, that's a HIGH issue.

---

## See Also

- [[01 - Subagents]]
- [[01 - Building Custom Agents]]
- [[03 - Multi-Agent Reviews]]
- [[08 - Git and PR Workflow]]
