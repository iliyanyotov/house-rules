---
name: small-changesets
description: Use when opening a PR, dividing work, or sizing a piece of in-flight code. Use when a PR exceeds ~400 lines of non-generated diff, touches three unrelated concerns, or accumulates "while I was in here" changes. Use when reviewing a PR that mixes a feature, a refactor, and a dependency bump.
---

# Small Changesets

## Overview

**A PR is one reviewable concern.** Target ≤400 lines of non-generated diff. If a reviewer can't read it carefully in under 30 minutes, it's too big.

Split refactor from behavior change; split feature from infrastructure; split each independently-shippable step into its own PR.

## When to Use

- Opening any PR
- Sizing in-flight work — *before* the diff exists
- A PR that mixes a feature, a refactor, and a dependency bump
- A "while I was in here" cleanup creeping into the change
- A PR description that starts with "this PR does several things"

## The Iron Rule

```
NEVER ship a PR that mixes unrelated concerns or exceeds ~400 lines.
```

**No exceptions:**
- Not for "it's all related"
- Not for "splitting is extra work"
- Not for "the reviewer will ask for context"
- Not for "our team merges everything as one squash anyway"

## Detection: The "Several Things" Smell

If the PR description needs more than three bullets, STOP and split:

```
❌ VIOLATION: bundled PR
  Title: "Add overdue-email feature and refactor invoice render and bump mailer dep"
  Diff:  +812 / -304
  Description:
    - Added new email-send job
    - Refactored InvoiceTemplate type
    - Bumped mailer library to v3
    - Cleaned up some legacy code while I was in there

✅ CORRECT: 4 stacked PRs, each independently reviewable
  PR 1 — chore: bump mailer to v3                       (~40 lines)
  PR 2 — refactor: extract InvoiceTemplate type         (~80 lines)
  PR 3 — feat: add overdue-email send (behind flag)     (~200 lines)
  PR 4 — chore: enable flag; delete legacy email path   (~40 lines)
```

Each PR is independently reviewable, shippable, and reversible.

## Why Small PRs Win

| Big PR | Small PR |
|---|---|
| Reviewers scan; defects slip past 400 lines | Reviewers actually read |
| Days to review; multiple stale-context round-trips | Hours to review; fresh context |
| Merge conflicts block both sides | Merges cleanly |
| `git bisect` returns "huge commit"; useless | `git bisect` lands on the actual change |
| Author re-explains scope in every reply | PR description holds the whole scope |
| Revert is risky — drags unrelated work with it | Revert is one PR |

## The Pattern

### Plan the split before writing the code

The cheapest split happens *before* the diff exists. Sketch the shape up front:

```
Goal: add overdue-invoice email feature

PR 1 — chore: bump mailer dep, wire env, smoke-test endpoint  (~50 lines)
PR 2 — refactor: extract InvoiceTemplate type from render fn  (~80 lines)
PR 3 — feat: add overdue check + email send, behind flag      (~200 lines)
PR 4 — chore: enable flag; delete legacy notification path    (~40 lines)
```

PR 1 lands without behavior change. PR 2 is pure tidying. PR 3 ships dark behind a flag. PR 4 finishes the rollout. Each PR has the previous as its base — reviewers approve in sequence; merges cascade.

### Already too big? Cherry-pick out

When you find yourself with 800 lines, *don't* push it as one. Reset your branch to the base, cherry-pick the tidying commits onto a new branch (PR 1), then cherry-pick the feature commits onto a branch based on PR 1's branch (PR 2). If commits mix concerns within a file, separate the commits first.

### "While I was in here" — open a follow-up

Found an unrelated smell while working? Don't fix it in the current PR. Note it in the description ("found unrelated issue #N, addressed separately") and open a follow-up. The reviewer of the current PR doesn't need to context-switch.

### What counts toward the budget

Production code, tests, and runtime config count. Lockfiles, codegen output, snapshot fixtures, and pure renames don't (within reason). When unclear, list bulk-changed files in the PR description with the reason.

## Pressure Resistance

### 1. "It's all related, it has to ship together"

**Pressure:** "If I split, the pieces don't make sense alone."

**Response:** "Has to ship together" and "has to merge together" are different. Stack the PRs. Ship behind a flag in PR 2; enable the flag in PR 3.

**Action:** Atomic *delivery* doesn't require atomic *commits*.

### 2. "Splitting is more work than just doing it"

**Pressure:** "I'd rather just push the big PR."

**Response:** Splitting *before* writing the code is planning, which you should do anyway. Splitting *after* is more work — but less work than the reviewer re-reading 800 lines twice.

**Action:** Plan the split up front. It saves time net.

### 3. "Reviewers don't want to context-switch between many small PRs"

**Pressure:** "Multiple small PRs are annoying for the reviewer."

**Response:** They *much* prefer many small PRs over one big one. The literature is unambiguous. Defect detection collapses past 400 lines.

**Action:** Ask a senior reviewer their preference. The answer is always small.

### 4. "Stacked PRs are complicated"

**Pressure:** "I don't know the stacked-PR workflow."

**Response:** First time, yes. Second time, less. By the third, it's a habit. Compared to an hour-plus review on a 1000-line PR, the 10-minute tooling learning curve is a bargain.

**Action:** Stack the PRs. Tools like Graphite or `git-spr` smooth the workflow.

## Red Flags

- A PR description with "this PR does X, Y, Z" or three+ bullets of different things
- Diff stat over 400 lines (excluding lockfiles and generated code)
- A title with "and" or commas separating concerns
- A PR open more than 3 working days with the author still pushing commits
- Review comments that cross-reference each other ("we'll address this in the refactor part below")
- The phrase "while I was in here" anywhere in the description
- A revert PR that has to manually re-apply unrelated changes from the reverted PR

**All of these mean: split before merging.**

## Quick Reference

| Situation | Action |
|---|---|
| PR > 400 lines | Split into stacked PRs |
| Mixed refactor + feature | Refactor first (PR 1), feature on top (PR 2) |
| "While I was in here" smell | Open a follow-up PR |
| Risky feature ready | Ship dark behind a flag; flip in a separate PR |
| Indivisible-looking change | Look for: plumbing → feature → rollout → cleanup |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|---|---|
| "It's all one logical unit" | Logical unit ≠ reviewable unit. Use stacked PRs. |
| "Reviewers will hate context-switching" | They prefer small PRs. The literature backs this. |
| "Splitting is extra git work" | About 10 minutes. You save 30+ in review iteration. |
| "I'm faster solo, splitting later" | "Later" is paid by the reviewer and the bug-finder. |
| "CI takes 20 minutes — I'd rather batch" | Then make CI faster. Don't compensate for slow CI with un-reviewable PRs. |
| "It's an experiment, who cares about the size" | Experiments either land or get deleted. If it lands, it gets reviewed. Split. |

## The Bottom Line

**One PR, one concern, under 400 lines, under 30 minutes to review.**

Plan the split *before* writing the code. Stack PRs when work has dependencies. Open follow-ups for "while I was in here" finds. Atomic delivery doesn't require atomic commits.

## Related

- `tidy-first-separate-commits` — split across concerns (PRs) vs. within (commits)
- `revert-and-redo-on-confusion` — when a PR spirals, revert and redo smaller

## Reference

- SmartBear / Cisco code review study (2008) — defect detection collapses past ~400 lines of diff. Still the canonical data on review size vs. defect-detection rate.
- Google, *Engineering Practices* — *"In general, the right size for a CL is one self-contained change."*
- Paul Hammant, [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/) — the trunk-based-development case for small, frequent merges.
- [Graphite](https://graphite.dev/), [git-spr](https://github.com/ejoffe/spr) — tooling for stacked PRs.
