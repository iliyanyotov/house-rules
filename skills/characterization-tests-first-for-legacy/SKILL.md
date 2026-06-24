---
name: characterization-tests-first-for-legacy
description: Use when about to refactor, fix a bug in, or extend a function/module you didn't write or don't fully understand. Use when "the original author is gone" or "we're not sure what this is supposed to do." Use when the existing code has no tests and you're about to add the first one. Use when tempted to "fix" code that doesn't have a failing test yet — the failing test must come first.
---

# Characterization Tests First, for Legacy

## Overview

**Before you change unfamiliar code, write tests that pin down its *current* behavior** — including the quirks you suspect are bugs. Only then refactor or extend. The new tests document what the code does today; *new* tests for the fix come after.

A "characterization test" doesn't ask *should this be true*. It asks *what is true right now*. You run the code, observe what it returns, and write the assertion that matches. The test becomes a tripwire: if your refactor changes the behavior, the test fails and you decide deliberately whether the change is intended.

## The Iron Rule

```
NEVER change unfamiliar code without first pinning its current behavior in tests.
```

**No exceptions:**
- Not for "writing tests for code that's already wrong is silly"
- Not for "it's faster to just fix the bug"
- Not for "I know what the code does"
- Not for "my PR will be twice as big with the tests"

## Why

Untested legacy code is opaque. Every line you change is a roll of the dice — you might fix the bug, you might break a feature whose tests don't exist. The fix becomes a discovery process that finishes when QA or production tells you what else broke.

The characterization test inverts that. You spend an hour up front discovering what the code does. The test suite that comes out is *ugly* — it asserts on weird edge cases, off-by-one quirks, suspicious-looking outputs. It documents the lived behavior, not the intended behavior. That's the point: now you can change the code, and the test tells you exactly what you altered. **The bug fix becomes a *deliberate* test failure, not a *surprise* production incident.**

This skill matters most when you've inherited a codebase, when you're maintaining code older than a few months without context, or when the original author has moved on. In a fresh greenfield project, you write TDD-style tests as you go. In a brownfield project, characterization tests come first.

## Detection

You are violating the rule if any of these are true:

- A bug fix lands without a test that *would have caught* the bug before the fix.
- A refactor lands on code that had no tests beforehand and no tests now.
- A PR description says "behavior should be the same" but no test asserts that.
- You're about to delete code "because nobody calls it" and you haven't checked or tested the assumption.
- You read a function, can't tell what it should do, and start typing changes.
- The phrase "I think this is what it does" appears in your head before "the test confirms this is what it does."

## The Pattern

### The basic cycle

```
1. Pick the smallest scope you can: one function, one method, one branch.
2. Call it with realistic inputs.
3. Observe what comes back.
4. Write a test asserting what comes back (even if it looks wrong).
5. Add tests until you've covered every branch you intend to touch.
6. Run the suite. It should pass — that's the lived behavior captured.
7. Now make your change. Watch which tests fail.
8. Decide deliberately: is each failure intended (the bug fix) or accidental (a regression)?
9. Update the test for intended failures; revert the code for accidental ones.
```

### Concrete: a quirky helper

Suppose you find this and need to "fix" it because it occasionally returns wrong values:

```ts
// src/utils/computeDiscount.ts
export function computeDiscount(price: number, code: string): number {
  if (code.startsWith('SAVE')) {
    return price * 0.9;
  }
  if (code === 'VIP') {
    return price - 10;
  }
  return price;
}
```

**Don't change it yet.** Write characterization tests first:

```ts
// src/utils/computeDiscount.test.ts
import { describe, expect, test } from 'vitest';
import { computeDiscount } from './computeDiscount';

describe('computeDiscount — current behavior (characterization)', () => {
  test('SAVE10 produces 10% off', () => {
    expect(computeDiscount(100, 'SAVE10')).toBe(90);
  });

  test('SAVE0 also produces 10% off', () => {
    // This is suspicious — is this intended? Capture the lived behavior anyway.
    expect(computeDiscount(100, 'SAVE0')).toBe(90);
  });

  test('VIP subtracts 10', () => {
    expect(computeDiscount(100, 'VIP')).toBe(90);
  });

  test('VIP on a $5 item returns -5', () => {
    // Probably a bug. Captured for later decision.
    expect(computeDiscount(5, 'VIP')).toBe(-5);
  });

  test('unknown codes return full price', () => {
    expect(computeDiscount(100, 'NOPE')).toBe(100);
  });

  test('empty string returns full price', () => {
    expect(computeDiscount(100, '')).toBe(100);
  });
});
```

Now you have a baseline. You suspect `VIP on $5` is a bug — your fix should change that test, not silently change behavior elsewhere. The other tests stay as the safety net.

The `(characterization)` label and the "suspicious / probably a bug" comments are *scaffolding for the discovery pass*, not a permanent fixture. Once you decide a quirk is intended, rename the test to state the real expectation; once you fix a confirmed bug, the changed test should read as an ordinary spec. Long-lived characterization suites end up looking like normal behavior tests named by intent — the "characterization" framing is how they start, not how they stay.

### Use a REPL or run the code to find weird inputs

The best characterization tests are written by *interacting* with the code: load it in a REPL, call it with values, see what happens. Running the test file with extra `console.log`s and reading output is another way. The point: don't guess what the code does — run it.

### Don't test what you don't intend to change

You don't need to characterize every function in the codebase. Characterize the scope you're about to modify, plus immediate callers if you're worried about ripple effects. Save the rest for when their turn comes.

### Snapshot tests for complex outputs

Prefer **explicit assertions** (`toEqual`, `objectContaining`) by default, even for complex objects — they document the intent ("these fields, these values") and fail with a readable diff. Pin a complex array-of-objects with `expect(rows).toEqual([objectContaining({ start, end, source })])`, not a blanket snapshot.

Reach for a **snapshot** only in two cases: (a) the output has many structurally-similar variants (e.g. a redactor checked across a dozen field-name shapes), where hand-writing each assertion is noise; or (b) the exact serialized form *is* the contract and writing it by hand is pure transcription. The snapshot *is* the assertion — read it, confirm it matches reality, commit it.

```ts
test('parseInvoice produces this exact shape today', () => {
  const result = parseInvoice(sampleInput);
  expect(result).toMatchSnapshot();
});
```

Warning: snapshots are easy to update mindlessly. Always *read* the diff before updating; don't just regenerate them blindly.

### Trace through callers before touching the function

The characterization tests don't have to be *only* on the function you're changing. If the function has 4 callers and you're not sure how the change affects them, characterize the callers too. Cheap insurance.

## Pressure Resistance

### "Writing tests for code that's already wrong is silly"

The test isn't endorsing the behavior — it's *documenting* it. The wrong-looking assertion is the evidence that the bug exists. When you fix the bug, that test changes; the others (the correct behaviors you preserved) stay green.

### "It's faster to just fix the bug"

Fast in the next hour. Slow in the next month when an unrelated thing breaks because you changed a side effect you didn't notice. The characterization test takes ~10 minutes; the regression hunt takes hours.

### "I know what the code does"

You think you do. Run it. Half the time the code behaves slightly differently from how it reads — closures captured the wrong scope, a falsy check matched `0`, an `await` was forgotten somewhere upstream. Lived behavior > read behavior.

### "There are no tests in this project"

That's why this skill exists. The characterization test is *the first test*. After this PR, there's one test. That's not less progress than starting with five tests; it's the right starting point.

### "The code is too coupled to test"

Then you have a seams problem, not a characterization problem. Solve the seam first (insert one parameter, one closure boundary, one injected dependency at the smallest possible place), then characterize.

### "My PR will be twice as big with the tests"

Yes. The tests can land as a separate PR — characterization first, then the fix. The reviewer sees: "PR 1: pin current behavior of X. PR 2: change X.foo to return Y when Z, updating the test that captured the old behavior." Each PR is small and the intent is unambiguous.

## Red Flags

- A bug-fix PR with no test added or changed.
- A "small refactor" PR touching a function that has zero existing tests.
- A code review comment "are you sure this doesn't break X?" with no test answering it.
- A commit message "fixed weird behavior" without a test asserting what was weird.
- The phrase "the old code was wrong, the new code is right" without tests proving both.
- A snapshot-update commit that updates >3 snapshots without explanation.
- Editing a function you've never seen before without first reading + running it.

**All of these mean: you're about to change unfamiliar code blind — pin its current behavior first.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's just a one-line fix" | One-line fixes ship regressions all the time, precisely because nobody thought a test was needed. |
| "Tests for ugly code are ugly" | Yes. That's a feature, not a bug — the ugliness shows what needs to change next. |
| "I'd be writing tests for behavior I'm about to change" | You're writing tests for the behavior that *stays*. The ones that change are how you confirm the fix landed. |
| "The original code didn't have tests, why should my change?" | Because you're touching it. The original author got away with no tests; you don't, because now there's a change of behavior to be deliberate about. |
| "It would take too long" | Half an hour, typically. Much less than the regression-hunt time you'll spend without it. |
| "Snapshot tests are cheating" | They're characterization, codified. The cheating is updating them without reading the diff. |

## Related

- `tdd` — the brownfield counterpart to test-first
- `seams-for-untestable-code` — add a seam first, then characterize

## Reference

- Michael Feathers, *Working Effectively with Legacy Code* (2004) — the canonical book on this discipline. Feathers' famous definition: **"legacy code is code without tests."** The characterization-test pattern is the entry point to changing legacy code safely.
- Kent Beck, *Test-Driven Development by Example* (2002) — covers the variant where you add tests to existing code by running it and recording the outputs.
- Martin Fowler, *Refactoring* (2nd ed., 2018) — every refactoring in the catalog assumes you have a green test suite. The first step before any refactor: "ensure you have a solid set of tests for this section of code."
