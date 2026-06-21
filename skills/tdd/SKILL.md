---
name: tdd
description: Use when implementing any new feature or function. Use when asked to "add tests later". Use when writing code before tests. Use when "this is hard to test" comes up — that's a design smell, not a testing smell.
---

# Test-Driven Development (TDD)

## Overview

**Write the test first. Watch it fail. Then write the code.**

TDD is not about testing — it's about design. Writing tests first forces you to think about the interface before the implementation.

## When to Use

- Implementing any new function, method, or feature
- Asked to "write code now, add tests later"
- Fixing a bug (write the test that reproduces it first)
- Refactoring existing code

## The Iron Rule

```
NEVER write production code without a failing test first.
```

**No exceptions:**
- Not for "it's a simple function"
- Not for "we'll add tests later"
- Not for "I already know how to implement it"
- Not for "tests slow me down"

The cycle: **RED** (write a failing test) → **GREEN** (write the minimum code to pass) → **REFACTOR** (clean up while tests stay green) → **REPEAT**. Each cycle is minutes, not hours.

## Detection: The Code-First Smell

If you're about to write implementation without a test, STOP:

```typescript
// ❌ VIOLATION: implementation first
function calculateShipping(weight: number, distance: number): number {
  return weight * 0.5 + distance * 0.1;
}
// "We'll add tests later if needed"

// ✅ CORRECT: test first
describe('calculateShipping', () => {
  it('charges $0.50 per kg plus $0.10 per km', () => {
    expect(calculateShipping(10, 100)).toBe(15);
  });

  it('returns 0 for zero weight and distance', () => {
    expect(calculateShipping(0, 0)).toBe(0);
  });
});

// NOW write the implementation.
function calculateShipping(weight: number, distance: number): number {
  return weight * 0.5 + distance * 0.1;
}
```

## Why Test-First Matters

| Test-After | Test-First |
|---|---|
| Verifies what the code does | Defines what the code should do |
| Implementation drives design | Requirements drive design |
| Tests often skipped | Tests always exist |
| Hard-to-test = poor design, discovered late | Hard-to-test = caught immediately |
| "Does it work?" | "Is it right?" |

## Pressure Resistance

### 1. "We'll add tests later"

**Pressure:** "Just write the function, we can test it after."

**Response:** "Later" never comes. Tests written after are weaker and often skipped entirely.

**Action:** Write the test now. It takes 30 seconds.

### 2. "It's too simple to test"

**Pressure:** "This function is trivial — testing is overkill."

**Response:** Simple functions grow complex. Simple bugs cause outages.

**Action:** Write the test. Simple functions get simple tests.

### 3. "I already know how to implement it"

**Pressure:** "The solution is already in my head."

**Response:** TDD isn't about uncertainty — it's about capturing requirements as executable specs.

**Action:** Write the test first. Prove the mental model is correct.

### 4. "Tests slow me down"

**Pressure:** "Writing tests takes extra time."

**Response:** Debugging takes 10× longer than testing. TDD is faster overall.

**Action:** The test you write now saves hours of debugging later.

### 5. "It's hard to test"

**Pressure:** "This function can't be tested without mocking five modules."

**Response:** That's not a testing problem — it's a design problem. The fix is "decompose the function," not "better mocking."

**Action:** Pull the pure logic out. Inject dependencies. Test the core, integration-test the shell.

## Red Flags

- Writing any function without a test
- "Let me just get it working first"
- Implementation file open without a test file
- Committing code without corresponding tests
- Tests that pass immediately (you never saw them fail)
- A test mocks more than one module per file
- A unit test talks to a real DB, network, or filesystem
- A function reads `Date.now()` inline and the test mocks the clock

**All of these mean: stop. Write the test first.**

## Quick Reference

| Situation | TDD Response |
|---|---|
| New feature | Write failing test → implement |
| Bug fix | Write test reproducing the bug → fix |
| Refactor | Ensure tests exist → refactor → tests pass |
| "Tests later" | Tests now. Always now. |
| "Hard to test" | Decompose first — pure logic out, deps in. |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|---|---|
| "Tests slow me down" | Debugging slows you down more. |
| "It's too simple" | Simple tests are fast to write. |
| "I'll test later" | You won't. Test now. |
| "I know it works" | Prove it with a test. |
| "Just a prototype" | Prototypes become production. Test them. |
| "TypeScript catches enough" | Types catch shape, not logic. Off-by-one passes the compiler. |

## The Bottom Line

**Test first. Code second. No exceptions.**

The test is the specification. The test is the documentation. The test is the safety net. Write it first, every time.

## Related

- `characterization-tests-first-for-legacy` — greenfield test-first vs. brownfield characterize-first
- `functional-core-imperative-shell` — TDD pressure yields the pure-core/shell split

## Reference

- Kent Beck, *Test-Driven Development by Example* (2002) — the canonical work; red-green-refactor.
- Kent C. Dodds, "The Testing Trophy" — *"The more your tests resemble the way the software is used, the more confidence they give you."*
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (2009) — "hard-to-test code is a design smell."
- Gary Bernhardt, "Functional Core, Imperative Shell" — the testable shape.
