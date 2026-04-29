---
tags: [claude-code, intermediate, workflow]
aliases: [TDD, Test-Driven Development, Red-Green-Refactor]
level: Intermediate
---

# TDD Workflow

> **One-liner**: TDD with Claude is *test-first prompting* — write the failing test, then ask Claude to implement *only enough* to make it pass. The discipline keeps Claude on rails.

---

## Quick Reference

| Phase | Goal | Prompt shape |
|-------|------|--------------|
| RED | Write a failing test for the next behaviour | "write a test that asserts X. Don't implement yet." |
| GREEN | Make the test pass with minimal code | "make the test pass. Smallest change. No refactoring." |
| REFACTOR | Clean up with tests still green | "refactor for clarity. Keep all tests passing." |

| Anti-pattern | Why it bites |
|--------------|--------------|
| "Implement feature X with tests" | Claude writes both at once → tests rubber-stamp the impl |
| Vague test ("test the function") | Claude invents requirements |
| Skip RED step | No proof the test actually catches the bug |
| Refactor + add features in same step | Hard to tell which change broke green |

---

## Core Concept

The point of TDD is not "have tests" — it's **forcing yourself to specify behaviour before writing code**. Used with Claude, it has a second benefit: the test is a *contract* Claude can't drift away from. Without a test, "implement pagination" can mean ten different things; with a test asserting `data.length === 20 && meta.total === 100`, the surface to satisfy is precise.

The cadence is small: one failing test, smallest implementation that passes, optional refactor, commit. Repeat. Big batches defeat the loop — if you let Claude write five tests and the implementation in one go, you've lost the steering wheel.

A good Claude TDD session also commits in the same cadence: `test:` commit on RED, `feat:` commit on GREEN, `refactor:` commit on REFACTOR. Three small commits beat one giant one.

---

## Syntax & API

### RED — write the failing test

```text
> Write a unit test for `paginate(items, page, limit)`:
  - returns the slice of items for the given page (1-indexed)
  - returns empty array if page is past the end
  - throws on limit < 1 or page < 1

  DO NOT implement the function yet. Just the tests + the function
  signature stub.
```

Run the test — it should fail (RED). If it passes accidentally (e.g. function already exists), step back.

### GREEN — implement minimally

```text
> Now implement `paginate` so the failing tests pass.
  Smallest change. No extra features, no edge cases beyond the tests.
```

Run the tests — green.

### REFACTOR — clean up

```text
> Refactor `paginate` for clarity (variable names, early returns).
  Run the tests after — they must still pass.
```

### Bug-fix variant — TDD on a real bug

```text
> The bug: `paginate(items, 0, 10)` returns the wrong slice.
  Step 1: write a test that fails for this exact case.
  Step 2: confirm the test fails.
  Step 3: fix the bug.
  Step 4: confirm green.
```

This is the canonical "regression test" pattern. The failing test *proves* the fix actually addresses the reported bug.

---

## Common Patterns

### Pattern: one behaviour per test

```text
> Add tests one at a time. After each test:
  1) run it — it must fail with a useful message
  2) implement the smallest change to pass
  3) run all tests — all green
  4) commit before moving to the next test

  Start with: empty input returns empty result.
```

### Pattern: TDD-guide subagent

If you have a `tdd-guide` agent installed (see [[01 - Building Custom Agents]]):

```text
> use the tdd-guide agent to add pagination to the users list endpoint.
  Strict RED-GREEN-REFACTOR. Three commits.
```

The agent enforces the cadence even when you'd be tempted to shortcut.

### Pattern: outside-in for a feature

```text
> Outside-in TDD for the new "export users to CSV" feature:
  1) E2E test: POST /users/export returns CSV with correct headers
  2) Integration test: handler calls service correctly
  3) Unit test: csv-formatting utility

  Top down. Get one level green before descending.
```

### Pattern: capture a hard-won bug as a test first

```text
> Yesterday's bug: timezone DST transition broke event scheduling.
  Before fixing — write a test that schedules an event during the
  spring-forward window and asserts the right time. The test should
  fail today. Only then fix.
```

### Pattern: enforce "tests first" via hook

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/require-test-first.sh" }
        ]
      }
    ]
  }
}
```

> The hook script can warn (not block) when an Edit touches `src/` without a recent matching test edit. See [[04 - Hooks]] and [[04 - Building Custom Hooks]].

---

## Gotchas & Tips

- **Don't ask for "tests + implementation" in one prompt.** You'll get tests that conform to the implementation, not specify it. Always RED first.
- **Always run the test after writing it.** A test you didn't see fail is not a real test — it might be passing by accident or not running at all.
- **Coverage isn't proof.** 100% line coverage with weak assertions is worse than 60% coverage with sharp ones. Tell Claude to assert *behaviour*, not just "function returns truthy."
- **Don't refactor without green tests.** If tests are red, you can't tell whether your refactor broke things or the test was already broken.
- **One test, one assertion family.** Multi-assertion tests obscure which behaviour actually broke. Group with `describe`, split tests by behaviour.
- **Mock at boundaries, not internals.** Mocking your own internal functions tests the mock, not the code. Mock external services (DB, HTTP) at their adapter.
- **Beware "I'll add tests after."** You won't, and Claude will lean on you to skip them. Make RED non-negotiable in the prompt.
- **For UI: integration over snapshot.** Snapshot tests are brittle and don't specify behaviour. Prefer "user clicks X, sees Y."
- **Run the full test suite, not just the changed file.** A change can pass its own test but break a neighbour.

---

## See Also

- [[09 - Code Review with Claude]]
- [[08 - Git and PR Workflow]]
- [[01 - Subagents]]
- [[09 - Common Workflows]]
