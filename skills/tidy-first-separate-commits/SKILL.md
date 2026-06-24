---
name: tidy-first-separate-commits
description: Use when about to refactor, rename, extract, reformat, or reorganize code in the same change as a behavior modification or bug fix. Use when staging a commit that includes both "I cleaned up X" and "I added/fixed Y." Use when reviewing a PR whose diff mixes whitespace, renames, and logic changes. Use when a commit message has "and" between two verbs.
---

# Tidy First, Separate Commits

## Overview

**Tidyings and behavior changes are different kinds of work, and they ship as different commits.** Never mix them in one commit.

A tidying *should* preserve behavior: rename, extract, inline, reformat, reorder. A behavior change *should* alter what the program does. The git log should be able to tell which is which at a glance.

## When to Use

- Staging a commit that mixes a refactor with a bug fix or feature
- A PR description that says "refactored X and added Y"
- A diff with whitespace/formatting changes interleaved with logic changes
- A reformat or lint sweep about to land in the same commit as a feature
- A rename across the project bundled with the work that prompted it

## The Iron Rule

```
NEVER mix a tidying with a behavior change in the same commit.
```

**No exceptions:**
- Not for "it's all one logical unit"
- Not for "splitting is a waste of time"
- Not for "I'll squash on merge anyway"
- Not for "the tidy was trivial"

## Detection: The "And" Smell

If the commit message needs the word "and" between a tidying and a change, STOP and split:

```
❌ VIOLATION
  "Rename computeFee to calculateFee and fix off-by-one in tier-3 pricing"

✅ CORRECT
  tidy(pricing): rename computeFee to calculateFee
  fix(pricing): off-by-one in tier-3 calculation
```

The first commit is *known safe*: same tests pass, same behavior. Reviewer skims, approves. The second commit is small: just the bug fix, on a clean foundation.

Seen in the wild: a commit titled *"remove console.log and improve type safety in tRPC routers"* bundled four classes of change behind one "and" — a security fix (removing logs that leaked private hashed-link data), a logger swap, a TS fix, and a Prisma `include`→`select` narrowing. Split:

```
fix(trpc): stop logging private hashed-link data
tidy(trpc): replace console.log with structured logger
tidy(trpc): drop @ts-expect-error, tighten types
perf(trpc): narrow duplicate include to select
```

The security fix in particular should never have ridden in behind "remove console.log" — buried in a cleanup commit, it's invisible to reviewers and to `git log --grep`.

## Why This Matters

On a **squash-merge** workflow the *PR* — not the individual commit — is the bisect/revert unit on `main`, so the `git bisect` / revert benefits below apply at *review time* (and on main only where commits survive the merge). That's not a reason to skip splitting: a clean split still makes review tractable, and a clear PR title/body becomes the durable record on main. Where commits *do* survive (merge or rebase workflows), the benefits extend to `main` too.

| Mixed commit | Split commits |
|---|---|
| Reviewer can't tell behavior change from rename | Reviewer sees behavior change isolated |
| 600-line diff with 30 lines of logic; gets rubber-stamped | 30-line behavior commit; gets actually read |
| `git bisect` flags "the rename or the bug fix?" — unclear | `git bisect` walks past known-safe tidies in one step |
| Revert drags unrelated tidying back to broken state | Revert touches one concern |
| Merge conflict has to untangle both classes of change | Tidy commit rebases cleanly; only the change conflicts |

## The Pattern

### Split a mixed working tree

```bash
# Stage only the tidying hunks.
git add -p src/pricing.ts
git commit -m "tidy(pricing): rename computeFee to calculateFee"

# Then stage the behavior change.
git add src/pricing.ts
git commit -m "fix(pricing): off-by-one in tier-3 calculation"
```

`git add -p` is the workhorse. If you can't tell hunks apart in your head, the commit was right to split.

### Already committed? Unstage and re-commit

```bash
# Uncommit, keep changes staged.
git reset HEAD~1 --soft
git reset                # unstage everything
git add -p               # rebuild the tidy commit
git commit -m "tidy: ..."
git add .                # stage the behavior change
git commit -m "fix: ..."
```

End state is the same; history becomes reviewable. Spend 90 seconds untangling.

### Larger restructurings ship as a series

```
tidy(db): extract Repository interface
tidy(db): add OrdersRepository, no callers yet
tidy(orders): migrate orderService to OrdersRepository
tidy(orders): delete dead direct-query helpers
feat(orders): add overdue-detection query (uses repository)
```

Each commit is independently revertable. `git bisect` walks past each tidy in one step. The final `feat:` commit is small — every prerequisite already in place.

### What counts as tidy vs. behavior

| Tidying (no behavior change) | Behavior change |
|---|---|
| Rename a symbol | Change what a function returns |
| Extract a function | Add a new code path |
| Inline a function | Modify validation logic |
| Reformat (prettier/biome) | Reorder side effects |
| Reorganize file structure | Change a default value |
| Add types to untyped code (no suppression removed, no runtime cast changed) | Add or remove a parameter; remove a `@ts-expect-error`/`@ts-ignore` or change a runtime cast |
| Sort imports | Add an env-var read |
| Add tests for existing behavior | Add a feature flag check |

When uncertain, ask: *would a reasonable test for the old code still pass after this change?* If yes, it's tidy. If no, it's behavior.

One trap for this test: **type-only changes are erased at runtime**, so the old test passes even when behavior shifted. Adding types is tidy *only* when no `@ts-expect-error`/`@ts-ignore` is removed and no runtime cast (`as`, `!`, a parse/coerce) changes. Removing a suppression can surface a real behavior change at a boundary — split it out as a `fix:`, not a tidy.

## Pressure Resistance

### 1. "It's all one logical unit"

**Pressure:** "The rename only makes sense because of the fix."

**Response:** A logical unit ships as a *series of commits in a single PR*. The PR is the unit; the commits are the audit trail. You don't need one commit to ship one feature.

**Action:** Open one PR with two commits. Reviewer reads them in order.

### 2. "Splitting is a waste of time"

**Pressure:** "Two minutes of `git add -p` for a 90-second tidy?"

**Response:** Splitting takes 90 seconds. Reviewing a mixed 800-line PR takes 30 minutes and yields lower confidence. Net cost is hugely negative.

**Action:** Split. The time pays back on the first review round-trip.

### 3. "Reviewers complain about too many commits"

**Pressure:** "Three commits for one feature looks like noise."

**Response:** The opposite — experienced reviewers prefer small commits because each is independently reviewable. The complaint is a junior-reviewer signal.

**Action:** Split. Add a PR description that walks through the commits in order.

### 4. "I'll squash on merge anyway"

**Pressure:** "It all becomes one commit on `main`."

**Response:** Then split for the review pass and squash on merge. Reviewers see the staged-out commits during review; `main` stays clean afterward. Best of both.

**Action:** Use both. Split commits during review; squash on merge.

### 5. "The tidy was trivial"

**Pressure:** "Why ceremony for a rename?"

**Response:** Triviality is exactly why it's a clean tidy commit. The reviewer scans, sees "rename," moves on in seconds.

**Action:** Commit it separately. The triviality is the win, not the excuse.

## Red Flags

- The word "and" in a commit message between a noun and a verb
- A commit's diff stat shows files unrelated to the change goal
- A PR diff with whitespace-only hunks adjacent to logic hunks
- A reformat or lint sweep committed together with a feature
- A rebase that consolidates unrelated work into one commit "for cleanliness"
- The phrase "I'll just squash these" used *before* the work is done, not after
- A revert that has to be hand-edited because the commit also contained good tidying

**All of these mean: split before pushing.**

## Quick Reference

| Situation | Action |
|---|---|
| Working tree has mixed tidy + change | `git add -p` to stage tidy; commit; stage rest; commit |
| Already committed mixed | `git reset HEAD~1 --soft`, then split |
| Big restructuring | Series of small tidies, each independently green |
| Refactor + new feature | Tidy first (one commit), feature on top (another) |
| Lint/format sweep | Its own commit, never with a feature |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|---|---|
| "Multiple commits clutter the log" | Squash on merge. The PR remains one commit on `main`; review history stays clean. |
| "I commit small but PR big" | This rule is about commits, not PRs. PRs can stay one. |
| "I can't tell what's tidy and what isn't" | Apply the test: "would the old test still pass?" |
| "The diff tool shows me both" | Diff tools show *that* lines changed, not which class of change. Commits are the classification. |
| "It's not worth the discipline for small changes" | Small changes are easiest to split. Discipline costs least there. |
| "Reverting is rare anyway" | Bisecting isn't. Splitting saves you the next time a bug surfaces. |

## The Bottom Line

**Tidy commits are known-safe. Change commits get the careful review. Mixed commits are neither.**

The cost of splitting is 90 seconds with `git add -p`. The cost of not splitting compounds over the codebase's life — every future bisect, every future revert, every future review.

## Related

- `small-changesets` — concern-splitting at commit vs. PR granularity
- `revert-and-redo-on-confusion` — muddled diffs reset by re-committing separately

## Reference

- Kent Beck, *Tidy First?* (O'Reilly, 2023) — structural changes and behavioral changes are economically and cognitively distinct, and should be sequenced separately.
- [Conventional Commits](https://www.conventionalcommits.org/) — the `tidy:` / `fix:` / `feat:` prefixes that make the classification visible in `git log`.
