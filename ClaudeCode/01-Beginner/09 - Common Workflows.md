---
tags: [claude-code, beginner, workflow]
aliases: [Workflows, Recipes]
level: Beginner
---

# Common Workflows

> **One-liner**: Reach for these recipes when you start a session — bug fix, feature add, refactor, code review, debugging — each has a proven prompt shape.

---

## Quick Reference

| Goal | Recipe |
|------|--------|
| Fix a bug | repro → diagnose → fix → test |
| Add a feature | scope → tests first → implement → verify |
| Refactor | plan mode → approve → execute → tests pass |
| Code review | `/review` or fork a `code-reviewer` agent |
| Debug a hang or crash | gather symptoms → form hypothesis → test it |
| Write a PR | summarize commits → draft body → push |
| Explain unfamiliar code | scope file or function → ask for "under 200 words" |

---

## Core Concept

Each workflow below is a **template** — copy, edit, paste. They all share the same structure: define what "done" looks like, give Claude the smallest sufficient scope, verify by running tests / reading the diff.

The biggest win from these templates is consistency. The same shape every time means you stop reinventing prompts and Claude's output gets predictable.

---

## Syntax & API

### Bug fix

```text
Bug: <one-sentence symptom>
Repro: <command, request, or test that demonstrates it>
Expected: <what should happen>

Fix in <file>. Add a test that fails before the fix and passes after.
```

### New feature

```text
Add <feature> to <area>.

Requirements:
- <bullet>
- <bullet>

Constraints:
- no new deps
- public API unchanged
- follow the structure of <existing similar feature>

Done when:
- new tests pass
- existing tests still pass
- <any specific verification>
```

### Refactor (use plan mode)

```text
> /permissions   # switch to plan mode

> Refactor <thing>. Goals:
  - <goal>
  - <goal>

  Plan first. List files + estimated risk. I'll approve before execution.
```

### Code review

```text
> /review                # if a PR / branch is open

# or, deeper review:
> Review the diff vs main. Focus on: correctness, security, missing tests.
  List up to 5 issues per category, ranked. No nitpicks.
```

### Debugging session

```text
Symptom: <what you observe>
Environment: <relevant facts — OS, runtime version, recent change>
What I tried: <previous attempts so we don't repeat>

Hypothesize 2-3 possible causes. For each, propose a quick check.
Run the cheapest check first.
```

### PR creation

```text
> Create a PR for the current branch.
  Title: ≤70 chars
  Body: summary (3 bullets) + test plan checklist
  Don't push if there are uncommitted changes.
```

### Explain unfamiliar code

```text
> Read <file>. In ≤150 words: what does it do, who calls it, any
  surprising behavior. Then list 3 questions a new contributor would have.
```

---

## Common Patterns

### Pattern: tight feedback loop on a flaky test

```text
> Run `npm test -- foo.test.ts` 5 times. Note which iterations fail
  and capture the failure text. Then hypothesize the source of flakiness.
```

### Pattern: explore-then-act in two prompts

```text
# turn 1 — exploration only
> Read src/api/auth and tell me how token refresh works. No changes.

# turn 2 — action with the model in your head
> The bug: refresh on a revoked token returns 200. Fix it in
  `auth.service.ts:refreshToken`. Add a test for the revoked case.
```

### Pattern: parallel forks for independent surveys

```text
> Run three forks in parallel:
  1) audit src/api/ for unhandled errors
  2) audit src/db/ for missing transactions
  3) audit src/jobs/ for unbounded retries
  Each fork: summary under 200 words.
```

### Pattern: "verify, then commit"

```text
> Make the change. Then:
  1) run the tests
  2) run typecheck
  3) only if both pass, stage and commit with a conventional message
```

### Pattern: short-circuit when context is wrong

```text
> stop. you're refactoring code I didn't ask you to touch.
  revert and re-read the original task.
```

---

## Gotchas & Tips

- **Don't chain unrelated tasks** in one session. Bug fix → feature → refactor in one chat = brittle. `/clear` between tasks.
- **Verify before committing.** Always run tests and typecheck. Claude can hallucinate that something works; the test runner can't.
- **A failing test after a "fix" is a fix that didn't work.** Don't accept "tests passed locally" without seeing the output.
- **Long debugging sessions get worse, not better.** If you've gone 30 turns without progress, stop, summarise, `/clear`, and start fresh with the summary as your prompt.
- **Code review is best done by a fork or sub-agent**, not the same conversation that wrote the code. Independent eyes find more.
- **For destructive workflows** (deletes, force-push, deploys) — even with a perfect prompt, demand the plan first.
- **PR creation should always start with a clean working tree.** Confirm via `git status` before you ask Claude to push.

---

## See Also

- [[07 - Effective Prompting]]
- [[08 - Plan Mode]]
- [[09 - Code Review with Claude]]
- [[10 - TDD Workflow]]
- [[08 - Git and PR Workflow]]
