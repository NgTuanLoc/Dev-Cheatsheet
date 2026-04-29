---
tags: [claude-code, intermediate, workflow]
aliases: [Refactoring, Large-scale Refactor, Codemod]
level: Intermediate
---

# Multi-file Refactoring

> **One-liner**: For changes that span more than a handful of files, **Plan mode first, then execute in small commits**. Don't trust a single shot to get a 50-file refactor right.

---

## Quick Reference

| Step | Tool / Approach |
|------|-----------------|
| 1. Audit scope | `Grep` / `gh search code` to count call sites |
| 2. Plan | Plan mode — write the playbook, no edits |
| 3. Execute | Small batches (5–10 files), commit per batch |
| 4. Verify | Type-check + tests after each batch |
| 5. Review | Fresh subagent, fork-review the diff |

| Refactor type | Best approach |
|---------------|---------------|
| Rename symbol | LSP rename if available; Claude only if cross-language |
| Reshape API (signature change) | Plan mode → adapter shim → migrate → remove shim |
| File reorganisation | Move first (no logic change), commit, then refactor |
| Codemod-able pattern | jscodeshift / ts-morph / comby — Claude writes the codemod |
| Type system migration (e.g. JS→TS) | One module at a time, gates tests at each step |

---

## Core Concept

The biggest failure mode in multi-file refactoring with Claude is **doing too much in one step**. Claude can produce a 200-file diff that *looks* correct, compiles, but breaks runtime behaviour in a corner case nobody asked about. By the time you find it, the diff is too big to bisect.

The discipline:

1. **Audit** the scope precisely — count files, list call sites, identify the riskiest ones.
2. **Plan** the migration in plan mode. The plan is the contract.
3. **Execute incrementally**. 5–10 files per batch, commit each, run tests each.
4. **Verify continuously**. Type-check + tests after every batch — failures stay local.
5. **Review with fresh eyes**. The agent that did the refactor is biased toward "looks fine."

For mechanical patterns (rename foo → bar everywhere), prefer a *codemod* (jscodeshift, ts-morph, comby) over freeform edits — Claude can write the codemod for you.

---

## Syntax & API

### Step 1 — audit scope

```text
> audit the impact of renaming the `User.email` field to `User.emailAddress`.
  - count call sites (grep `\.email\b` in TS/JS/SQL/templates)
  - separate read sites from write sites
  - list any external surfaces (API responses, DB column, GraphQL schema)
  - estimate effort per layer

  Don't propose changes yet. Just the audit.
```

### Step 2 — plan in plan mode

Enter plan mode (Shift+Tab or `/plan`):

```text
> Plan the rename `User.email` → `User.emailAddress`:
  - back-compat strategy (alias the field for one release?)
  - migration order: DB → API → consumers
  - test strategy per layer
  - rollback plan if a layer fails

  Output: numbered playbook with commit titles.
```

Claude returns a plan. Read it. Refine it. Approve via ExitPlanMode.

### Step 3 — execute in batches

```text
> Step 1 of the plan: add `emailAddress` as an aliased getter on User
  that delegates to `email`. Don't touch any callers yet.
  Run tests. Commit "refactor: alias emailAddress on User".
```

Then:

```text
> Step 2: migrate the `services/` folder to use `emailAddress`.
  Files only — no DB or API changes yet. Test, commit per file or per
  small group. Stop after 10 files for a check-in.
```

### Step 4 — verify

After each batch:

```bash
pnpm typecheck && pnpm test
git status   # confirm staged scope matches plan
```

If green, commit. If red, fix or rollback that batch (`git reset --hard HEAD`) before proceeding.

### Step 5 — fresh-eyes review

```text
> Spawn a fork to review the full refactor diff (`git diff main...HEAD`).
  Focus: did any call site get missed? Are there string-based references
  (logs, templates, JSON keys) that grep didn't catch? Tests still pass —
  but is behaviour preserved?
```

---

## Common Patterns

### Pattern: codemod-first for mechanical changes

```text
> Write a jscodeshift codemod that renames calls of `oldFn(x)` to
  `newFn({arg: x})`. Run on `src/`. Show me the diff stats first;
  I'll review before applying.
```

A codemod is auditable, deterministic, and re-runnable. Beats a 50-message conversation that drifts.

### Pattern: adapter shim for API reshape

For a signature change `oldFn(a, b)` → `newFn({a, b, c})`:

1. Add `newFn` alongside `oldFn`. `oldFn` delegates to `newFn` with default `c`.
2. Migrate callers one batch at a time.
3. Once all callers are migrated, delete `oldFn`.

This way you never have a broken master branch.

### Pattern: move-then-refactor

To split a file or reorganise structure:

```text
> Step 1: move `src/utils/helpers.ts` to `src/utils/string-helpers.ts`,
  `src/utils/array-helpers.ts`, `src/utils/object-helpers.ts`.
  No content changes — just move + update imports.
  Commit "refactor: split helpers.ts by category".
```

Then in a *separate* commit, rename functions or restructure. **Mixing move and edit in one commit makes review impossible** — git can't tell what's a move vs a change.

### Pattern: language migration in slices

JS → TS:

```text
> migrate one module at a time. Order: leaf modules first.
  After each module:
  - rename .js → .ts
  - add types only where TS demands; don't perfect what was already implicit
  - run typecheck + tests
  - commit
  Stop after 5 modules so I can review.
```

### Pattern: rollback gate

```text
> after each batch: if typecheck or tests fail, STOP. Don't try to fix.
  Report the failing diagnostic; we'll decide whether to rollback.
```

This prevents Claude from "fixing" a small failure by digging deeper into a wrong refactor.

---

## Gotchas & Tips

- **Don't ask for "the whole refactor" in one prompt.** Plan + execute is two prompts minimum, usually many more.
- **Grep misses string-based references.** Log messages, JSON keys, GraphQL schema, SQL column aliases. Audit each layer separately.
- **Renames are not just text.** Database columns, env var names, log fields, metric names — each has its own rename ritual.
- **Type checks are necessary but not sufficient.** A refactor that type-checks may still break runtime (different default behaviour, lost side-effect order). Tests matter.
- **Don't squash a multi-day refactor into one commit.** Bisect becomes impossible if a regression appears later.
- **Claude tires on long refactors.** Context fills with previous edits. Reset the session between batches if accuracy drops.
- **Test the perimeter, not the centre.** After a sweeping refactor, verify the *boundary behaviour* (API responses, DB writes, UI flows) — not just unit tests.
- **For codemods, dry-run first.** `jscodeshift -d` shows what would change without writing.
- **A migration with a "remove the shim later" step is unfinished.** Use [[11 - Background Tasks]] `/schedule` to leave a reminder.
- **Worktrees help when the refactor will run unattended** — see [[12 - Worktrees and Parallel Work]].
- **For renaming within a single language, use the IDE's LSP rename first.** It's structural; Claude is textual. Reserve Claude for cross-language or pattern-based refactors.

---

## See Also

- [[08 - Plan Mode]]
- [[12 - Worktrees and Parallel Work]]
- [[09 - Code Review with Claude]]
- [[10 - TDD Workflow]]
