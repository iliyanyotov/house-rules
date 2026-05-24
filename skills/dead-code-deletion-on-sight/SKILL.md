---
name: dead-code-deletion-on-sight
description: Use when finishing any change that left a function, type, file, import, env var, feature flag, or commented-out block unused. Use when reviewing a PR that introduces "just in case" code or leaves old code paths beside new ones. Use when grep-finding zero-reference symbols.
---

# Dead Code, Deletion on Sight

## Overview

**If a symbol has zero references after a change, delete it in the same PR.** Never keep code "just in case." The codebase is not a museum.

Git remembers. Deletion is reversible. Keeping unused code is not free — it's a slow tax on every future reader.

## When to Use

- Finishing any change that left a symbol with zero references
- Reviewing a PR that introduces "just in case" branches
- A feature flag that's been at 100% for two weeks — and the inactive branch still ships
- A commented-out block that's been there longer than a sprint
- A `@deprecated` JSDoc tag with no removal date and no current consumers

## The Iron Rule

```
NEVER keep unused code. Delete on sight; git remembers if you need it back.
```

**No exceptions:**
- Not for "we might need it again"
- Not for "it took effort to write"
- Not for "I'll delete it next sprint"
- Not for "tests depend on it" (the tests are also dead)

## Detection: The Zero-Reference Smell

If a symbol has zero references and the build still succeeds, STOP and delete:

```ts
// ❌ VIOLATION: feature flag at 100% for weeks, both paths still ship.
export async function createOrder(input: OrderInput) {
  if (await flags.isOn('use_v2_order_path')) {
    return createOrderV2(input);
  }
  return createOrderV1(input);  // dead since the flag flipped
}

// ✅ CORRECT: confirm the flag is at 100%, then delete the branch
// AND the flag AND the V1 implementation in the same PR.
export async function createOrder(input: OrderInput) {
  return createOrderV2(input);
}
```

The flag exists to support a decision *that has been made*. Leaving it in place inverts the cost — flags accumulate, the system gets less testable, not more.

## Why Dead Code Costs

| Dead code looks free | Dead code actually costs |
|---|---|
| Build still passes | Misleads `grep` — two `formatInvoice` results, neither obviously the live one |
| Tests still pass | Distorts refactors — rename or signature change drags the dead version |
| "It compiles" | Carries security debt — dead endpoint or env var still exists at the boundary |
| "Tree-shaking removes it" | Tree-shaking doesn't reach dead types, dead branches, dead test fixtures |
| "Just a few extra lines" | Misleads new contributors — they read it, assume it matters, build on top |

Every line of code is a maintenance commitment. Code you aren't using is commitment without benefit.

## The Pattern

### Reactive — every PR includes a deletion pass

After your change, run "find references" on every symbol you stopped calling. If a symbol drops to zero references, the definition goes in the *same* PR — not a follow-up, not a TODO, not an issue.

### Proactive — periodic sweep

Quarterly or monthly, run an unused-export checker and delete what it finds. Process the report in one PR per category (unused exports, unused files, unused deps). Each is reviewable in minutes because the diff is pure deletion.

### Feature flags — delete the branch *and* the flag together

When a flag has been at 100% for two weeks, delete the inactive branch in the same release that confirms stability. Delete:
- The old code path
- The flag check
- The flag's definition
- Any flag-management code referencing it

The flag's job is done. Leaving it in place inverts the cost.

### Commented-out code — delete unconditionally

```ts
// ❌ Always wrong.
function compute(x: number) {
  // const old = legacyCompute(x);
  // if (old !== x * 2) console.warn('mismatch', x, old);
  return x * 2;
}

// ✅
function compute(x: number) {
  return x * 2;
}
```

If the commented-out code might be useful later, `git log` is one command away. The current file should reflect the current state.

### Backward-compatibility shims with no callers

```ts
// ❌ "Kept for backward compatibility" — and zero current consumers.
/** @deprecated use formatPriceCents */
export function formatPrice(cents: number) {
  return formatPriceCents(cents);
}

// ✅ Confirm zero consumers, delete.
// (If the symbol is part of a published external API, that's a different
// rule — internal deprecations don't apply.)
```

The shim exists *for callers*. No callers, no shim.

## Pressure Resistance Protocol

### 1. "We might need it again"

**Pressure:** "What if we want this code back next quarter?"

**Response:** You won't. And if you do, `git log -S 'formatInvoice'` finds it in two seconds. The future cost of resurrection is one command; the ongoing cost of carrying dead code is paid by every reader, every grep, every refactor.

**Action:** Delete now. Git holds the history.

### 2. "It took effort to write, deleting feels wasteful"

**Pressure:** "I worked hard on this."

**Response:** Sunk cost. The effort is already paid. The choice now is: pay more (maintenance) or stop paying (delete). The work isn't lost — git holds it.

**Action:** Delete. The effort was about learning, not about preserving the artifact.

### 3. "I'll delete it next sprint"

**Pressure:** "I'll come back to clean up later."

**Response:** Next sprint never comes. The PR that *created* the dead code is the only PR where the context is fresh. Bundling deletion is cheap; coming back to it cold is expensive and rarely happens.

**Action:** Delete in the same PR that orphaned it.

### 4. "Other code might still reference it that I haven't seen"

**Pressure:** "What if something I don't know about still uses this?"

**Response:** That's what "Find References" is for. If your IDE shows zero, your tests still pass, and the build succeeds — there are no callers.

**Action:** Run "Find References." If zero, delete.

### 5. "Tests depend on the dead code"

**Pressure:** "I can't delete it — there's a test for it."

**Response:** If the only consumer is the test, the test is also dead. A test of dead code is dead code with a `.test.ts` suffix.

**Action:** Delete both.

## Red Flags — STOP and Reconsider

- An IDE warning "X is declared but never used"
- A commented-out block of code longer than one line
- A file beginning with `// LEGACY` or `// DEPRECATED` and no removal date
- A `@deprecated` JSDoc tag with no removal version
- A feature flag definition older than 90 days
- An env var present in `.env.example` but referenced nowhere in code
- A dependency in `package.json` that an audit can't find any import for
- The phrase "let me leave it just in case"

**All of these mean: delete now, in this PR.**

## Quick Reference

| Situation | Action |
|---|---|
| Symbol with zero references after a change | Delete in the same PR |
| Feature flag at 100% for 2+ weeks | Delete the old branch, the check, and the flag |
| Commented-out block | Delete unconditionally; `git log` holds it |
| Unused export (knip / ts-prune / depcheck) | Delete in a category-PR |
| `@deprecated` internal API with no callers | Delete |
| Test exists for a function nothing else calls | Delete both |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|---|---|
| "Git won't remember" | Git remembers everything. `git log -p`, `git log -S`, blame, reflog. |
| "Deletion is risky" | Less risky than leaving misleading code that a future contributor builds on. |
| "It's harmless to keep" | Dead code is *visible* to readers. Visible code is implicitly endorsed. |
| "Someone else owns it" | Their unused code is your slow read. Open a PR or a quick chat; don't perpetuate. |
| "It's referenced in docs/comments" | Update or delete the docs/comments. They're equally stale. |
| "We'll do a cleanup sprint" | No team has ever successfully cleaned up at the end. The cleanest codebase is the one cleaned continuously. |

## The Bottom Line

**Zero references = delete now. Git holds the history if you ever need it back.**

The PR that orphans the code is the only PR where the context is fresh enough to delete it cleanly. Wait, and the deletion never happens.

## Reference

- Martin Fowler, *Refactoring* 2e (2018) — "Remove Dead Code" listed as a primary refactoring.
- Robert C. Martin, *Clean Code* (2008) — the boy-scout-rule framing: leave the campsite cleaner than you found it.
