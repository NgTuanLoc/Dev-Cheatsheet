---
tags: [claude-code, intermediate, workflow]
aliases: [Git, PR, Pull Request]
level: Intermediate
---

# Git and PR Workflow

> **One-liner**: Claude is excellent at commits, branches, and PRs — but you decide *when* it pushes, force-pushes, or creates remote artefacts. Use the conventional patterns below.

---

## Quick Reference

| Task | Prompt shape |
|------|--------------|
| Commit current changes | "commit with a conventional message" |
| Squash WIP commits | "squash the last N commits into one with message X" |
| Rebase on main | "rebase onto main, resolve conflicts conservatively" |
| Open a PR | "create a PR for this branch" (uses `gh`) |
| Review a PR | `/review <PR#>` |
| Find related commit | "git log -S 'symbol'" |

| Convention | Format |
|------------|--------|
| Commit type | `feat / fix / refactor / docs / test / chore / perf / ci` |
| Commit | `<type>: <description>` (≤ 72 chars subject) |
| Branch | `<type>/<short-slug>` (e.g. `feat/user-pagination`) |
| PR title | ≤ 70 chars; details in body |

| Risky op | Default |
|----------|---------|
| `git push --force` to main | **Always confirm explicitly** |
| `git reset --hard` | Always confirm |
| `--no-verify` | Don't use unless asked |
| Force-push to PR branch | OK after rebase, but say so |

---

## Core Concept

Git operations are routine but **destructive failure modes are real**. Claude follows a default safety posture — confirms before force-push, doesn't skip hooks, prefers new commits over amends. The trade-off is occasional friction; the benefit is no lost work.

The right pattern: do small commits often, let Claude draft messages following your repo's convention, and use `/review` (or a code-reviewer subagent) before opening a PR. For big changes, plan-mode-first then execute.

PR creation uses `gh` (GitHub CLI). Make sure it's installed and authed (`gh auth status`).

---

## Syntax & API

### Stage and commit

```text
> commit the current changes with a conventional message
```

Claude runs `git status` + `git diff`, drafts a message, commits. (You'll see the message before it commits — Claude follows the safer "ask first if unclear" rule.)

### Commit only specific files

```text
> commit only src/api/users.ts and its test, leave everything else.
  message should focus on the why.
```

### Squash WIP into one commit

```text
> squash the last 3 commits into one. Title: "feat: add user pagination"
  Body: 3-bullet summary.
```

### Conventional commit format

```text
feat: add pagination to GET /users

- adds page/limit query params with sensible defaults
- preserves response shape, adds `meta.total`
- new tests cover empty/last-page edge cases
```

### Open a PR

```text
> create a PR for this branch
```

Claude runs `git log main..HEAD` + `git diff main...HEAD`, drafts a title + body with summary and test plan, then `gh pr create`. You see the URL.

### Custom PR template

```text
> create a PR. Title under 70 chars. Body sections:
  ## Summary (3 bullets)
  ## Why
  ## Test plan (checkbox list of what to verify)
  ## Risks
```

### Review a PR

```text
> /review 1234           # GitHub PR #1234

# Or, for the local branch:
> /review

# Or via subagent:
> have the code-reviewer agent review this branch
```

---

## Common Patterns

### Pattern: TDD commit cadence

```text
> Step 1: write a failing test for the empty-result case. Commit "test: ...".
> Step 2: implement. Commit "feat: ...".
> Step 3: refactor. Commit "refactor: ...".
```

Three small commits beat one giant "feat: pagination + tests + cleanup."

### Pattern: stack the safety net

Project `<project>/.claude/settings.json`:

```json
{
  "permissions": {
    "deny": [
      "Bash(git push --force:*)",
      "Bash(git push --force-with-lease origin main:*)",
      "Bash(git push origin main:*)"
    ]
  }
}
```

> Even if Claude tries to push to main, the deny rule catches it. Confirms intent.

### Pattern: pre-PR ritual

```text
> Before opening the PR:
  1) git status — must be clean
  2) git pull --rebase origin main
  3) npm test && npm run typecheck
  4) /review my own diff
  5) only then create the PR
```

### Pattern: address review comments

```text
> The reviewer left 3 comments on PR #1234:
  - <comment 1>
  - <comment 2>
  - <comment 3>

  Address them. New commit per comment, conventional messages.
  Don't force-push.
```

### Pattern: rebase + force-push (with care)

```text
> rebase this branch onto main. After resolving conflicts, force-push
  with --force-with-lease, NOT --force.
  If the rebase has more than 5 conflicts, stop and report.
```

---

## Gotchas & Tips

- **Claude prefers new commits to amends.** Amending a pushed commit changes its hash and breaks reviewers. Only amend before push.
- **`--force-with-lease` ≠ `--force`.** The former refuses if the remote has new commits; the latter clobbers. Always prefer `--force-with-lease`.
- **Don't blanket-allow `Bash(git push:*)`** — it allows pushing to any remote/branch including main.
- **PR creation requires `gh`** authed in your shell. `gh auth status` should show "Logged in to github.com".
- **Long commit messages**: pass via heredoc; Claude does this automatically. Inline `-m "..."` breaks on shell quoting.
- **Don't let Claude split unrelated changes into one commit.** Ask for separate commits per concern.
- **Don't review your own PR with the same Claude session that wrote it** — it's biased. Use a fresh subagent or fork.
- **`gh pr create` opens a draft if you say so** — ask explicitly: "create the PR as a draft."
- **`git --no-verify` skips hooks** — never default. If a pre-commit hook fails, fix the underlying issue.

---

## See Also

- [[09 - Code Review with Claude]]
- [[10 - TDD Workflow]]
- [[09 - Common Workflows]]
- [[05 - Permissions and Safety]]
