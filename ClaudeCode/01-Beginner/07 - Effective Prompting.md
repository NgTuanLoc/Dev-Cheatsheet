---
tags: [claude-code, beginner, prompting]
aliases: [Prompting, How to Ask, Prompt Patterns]
level: Beginner
---

# Effective Prompting

> **One-liner**: Specific scope + concrete pointers (file paths, function names) + clear "done" criteria gets you 10× the result of "fix this please."

---

## Quick Reference

### Anatomy of a good prompt
| Part | Why it matters | Example |
|------|---------------|---------|
| **Goal** | Tells Claude what "done" means | "Add pagination to GET /users" |
| **Scope** | Bounds the change | "Touch only `users.controller.ts` and its test" |
| **Pointers** | Skips discovery | "See existing pagination on `posts.controller.ts:42`" |
| **Constraints** | Avoids surprise | "No new deps. Keep response shape identical for backwards-compat." |
| **Done criteria** | How you'll verify | "All tests pass. New test for page=2 case." |

### Prompt antipatterns
| Bad | Why | Better |
|-----|-----|--------|
| "Fix the bug" | No bug specified | "Login returns 500 when password has `>` — repro: `curl … -d 'password=a>b'`" |
| "Make this better" | No metric | "Reduce cyclomatic complexity in `parseDate` — currently 14, target ≤ 8" |
| "Refactor everything" | Unbounded scope | "Refactor `users.service.ts` only. Don't touch tests yet." |
| Pasting a 500-line file | Wastes context | "Read `src/big-file.ts` and look at `processOrder`" |

---

## Core Concept

Prompting Claude is closer to **task delegation** than chat. Treat it like writing a ticket for a competent colleague who hasn't seen the issue: enough context to act, not so much that you're explaining what they already know.

The two failure modes are **under-specifying** ("fix this") and **over-specifying** (writing an essay when one sentence would do). The middle is: state the goal, point to the relevant code, list the constraints, define done.

When the task is exploratory ("what should we do about X?"), say so — Claude will respond with a recommendation, not jump to implementation. When it's a clear instruction, say *that* — and Claude will execute.

---

## Syntax & API

### Pattern: scoped change

```text
Add a `--dry-run` flag to the migration script in `scripts/migrate.ts`.

When `--dry-run` is set:
- log the SQL but do not execute it
- exit code 0 if SQL is valid, 1 if any error

Don't change anything else in the file.
Add a test in `scripts/migrate.test.ts` covering both paths.
```

### Pattern: bug report → fix

```text
Bug: GET /api/users/:id returns 500 when id is not a number.
Repro: `curl localhost:3000/api/users/abc`
Expected: 400 Bad Request with `{ error: "id must be numeric" }`

Fix in `users.controller.ts`. Add a test.
```

### Pattern: research, no implementation

```text
Don't change any code. Read `src/api/auth/*` and tell me:
1. Where do we verify JWTs?
2. What library?
3. Are there code paths that skip verification?

Keep the answer under 200 words.
```

### Pattern: constrained refactor

```text
Refactor `parseInvoice` in `src/billing.ts` to be testable:
- extract pure logic from I/O
- keep the public signature the same (callers must not change)
- add tests for the extracted pure functions
- no new deps
```

### Pattern: explicit "done" criteria

```text
Task: cache the result of `getUserPermissions`.

Done when:
- second call within 60s returns cached result (test it)
- cache invalidates on `updateUserPermissions` (test it)
- existing tests still pass
- no new external deps (use `Map` + timestamp)
```

---

## Common Patterns

### Use file paths and line numbers

```text
# good — Claude knows exactly where to look
> the bug is in `src/db/pool.ts:118` — the timeout is wrong

# bad — Claude has to search
> the bug is somewhere in the db code
```

### Reference existing code as a template

```text
> Add a /metrics endpoint, modeled on the existing /health endpoint
  in `src/api/health.ts`. Same module structure, same test layout.
```

### Tell Claude what *not* to do

```text
> Add the new feature, but:
  - don't change the public API
  - don't add new top-level folders
  - don't introduce a new HTTP client (we use undici)
```

### Ask for the plan first when stakes are high

```text
> Before changing anything, sketch the plan: what files, what shape,
  what tests. I'll approve before you implement.
```

(See [[08 - Plan Mode]] for the formal version.)

### Bound the context

```text
# bad — open-ended, may pull in unrelated files
> review the codebase for problems

# good — bounded
> review src/api/orders/ only. List up to 5 issues, ranked by severity.
```

---

## Gotchas & Tips

- **Don't paste large files** unless you must — Claude can `Read` them. Pasting wastes context.
- **Don't repeat what's in `CLAUDE.md`.** It's already loaded.
- **One concern per prompt.** Bug fix + refactor + new feature in one message = sloppy result.
- **"Why" beats "what" for non-obvious decisions.** "Why are we caching here?" → Claude reasons about it. "What is caching?" → wasted turn.
- **Use exact identifiers.** `userId` and `user_id` are different strings — say which.
- **Long sessions drift.** When the topic shifts, `/clear` and start fresh. Carrying 50 turns of irrelevant context hurts quality and costs more.
- **Push back on bad assumptions.** If Claude infers something wrong, correct early — small mid-course corrections beat undoing 10 minutes of work.
- **Phrases that move the needle**: "be terse", "don't add error handling for cases that can't happen", "don't refactor surrounding code", "report under 200 words", "explain your reasoning briefly".

---

## See Also

- [[08 - Plan Mode]]
- [[06 - CLAUDE.md Files]]
- [[10 - Tips and Pitfalls]]
- [[10 - Prompt Engineering for Code]]
