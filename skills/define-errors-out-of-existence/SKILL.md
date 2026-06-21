---
name: define-errors-out-of-existence
description: Use when designing an API, function signature, or library boundary. Use when about to throw an exception for a common, non-exceptional input. Use when a caller will immediately wrap your call in `try`/`catch` to handle a "normal" case. Use when the error case is more common than the success case.
---

# Define Errors Out of Existence

## Overview

**If a function throws for inputs the caller can reasonably produce, the function signature is wrong.** Reshape the API so the error case becomes a legal, typed return — not an exception.

Exceptions are for *exceptional* conditions. "User wanted to substring past the end of a string" is not exceptional. "Database is unreachable" is.

## The Iron Rule

```
NEVER throw for inputs the caller can reasonably produce. Encode the case in the return type.
```

**No exceptions:**
- Not for "throwing is shorter to write"
- Not for "throwing is more idiomatic in JavaScript"
- Not for "result types don't compose well"
- Not for "I'd have to refactor every caller"

## Why

Every `throw` for a common input pushes the burden onto every caller. The caller has to know to wrap the call, has to name a catch variable, has to pattern-match on it, and has to remember the *next* time they call it. Multiply by every consumer; you've turned a one-line API decision into a sprawl of defensive code.

The alternative isn't "swallow the error." The alternative is **make the error case impossible by design** — by returning a sensible value, by narrowing inputs so the bad case can't be constructed, or by returning a discriminated `{ ok, ... }` result.

Defensive programming says "guard every call." This rule says "guard at the design boundary, then trust."

## Tension with fail-fast

These two rules describe opposite sides of one decision, and both apply at different boundaries:

- **define-errors-out-of-existence** applies to **expected inputs from outside** — user-supplied data, third-party responses, form values, optional resources. The error case is *normal*; return a tagged result or a sensible value.
- **fail-fast** applies to **invariants you control** — env vars, function preconditions, internal contracts, programmer assumptions. The "error" is *abnormal*; throw immediately.

When reviewing a `catch` block: if the *caller* is the UI or an external system, returning `[]` / `null` / `{ ok: false, ... }` is correct here. If the *caller* is internal code that promised valid input, throwing is correct under fail-fast.

The two rules don't disagree — they describe what to do at different boundaries.

## Detection

You are violating the rule if any of these are true:

- A function throws for input that a UI form or query string could legally produce.
- Every (or nearly every) call site wraps the function in `try` / `catch`.
- The function's JSDoc lists a "throws" clause for an input-driven condition.
- A caller computes a check (`if (i >= arr.length) ...`) immediately before calling the function — that's the function's own contract leaking.
- The function's "happy path" is shorter than its "things that could go wrong" list.
- The phrase "the caller is responsible for ensuring..." appears in a docstring.

## The Pattern

### Ousterhout's canonical example — `substring`

```ts
// ❌ Throws for "out of bounds" — extremely common input.
function substring(s: string, start: number, end: number): string {
  if (start < 0 || end > s.length || start > end) {
    throw new RangeError('out of bounds');
  }
  return s.slice(start, end);
}

// ✅ Define the error out of existence — clamp to the legal range.
function substring(s: string, start: number, end: number): string {
  const a = Math.max(0, Math.min(start, s.length));
  const b = Math.max(a, Math.min(end, s.length));
  return s.slice(a, b);
}
```

JavaScript's built-in `String.prototype.slice` already does this. That's not an accident — the language designers applied the same rule.

### Lookup with a missing key — return a default, not throw

```ts
// ❌ Every caller wraps this.
function getRole(roles: Record<string, RoleDef>, key: string): RoleDef {
  const role = roles[key];
  if (!role) throw new Error(`Unknown role: ${key}`);
  return role;
}

// ✅ Encode the absence in the type.
function getRole(roles: Record<string, RoleDef>, key: string): RoleDef | undefined {
  return roles[key];
}

// Or, if a default makes sense:
function getRole(
  roles: Record<string, RoleDef>,
  key: string,
  fallback: RoleDef,
): RoleDef {
  return roles[key] ?? fallback;
}
```

The `undefined` return forces the caller to acknowledge the case via the type system — without forcing them to write a `try` / `catch`.

### Form submission — return a tagged result, don't throw for validation

```ts
// ❌ Throws on every validation failure. Every form caller needs try/catch.
async function createOrder(input: unknown) {
  const parsed = OrderInput.parse(input); // throws on bad input
  return orders.create(parsed);
}

// ✅ The "user typed something wrong" case is not exceptional. Return it.
type Result<T, E = string> = { ok: true; data: T } | { ok: false; error: E };

async function createOrder(input: unknown): Promise<Result<Order>> {
  const parsed = OrderInput.safeParse(input);
  if (!parsed.success) return { ok: false, error: parsed.error.message };

  const order = await orders.create(parsed.data);
  return { ok: true, data: order };
}
```

The caller pattern-matches on `result.ok` instead of catching. Exceptions are reserved for "database is on fire," which is genuinely exceptional and *should* propagate to the global handler.

### Narrowing inputs so the bad case can't exist

```ts
// ❌ Takes any number, throws if it's not a non-negative integer.
function setRetries(n: number): void {
  if (!Number.isInteger(n) || n < 0) throw new RangeError('expected non-negative integer');
  /* ... */
}

// ✅ Restrict the type. Bad inputs are a compile error, not a runtime throw.
type NonNegativeInt = Brand<number, 'NonNegativeInt'>;
const NonNegativeInt = {
  parse: (n: unknown): NonNegativeInt => {
    if (!Number.isInteger(n) || (n as number) < 0) {
      throw new RangeError('expected non-negative integer'); // parser is the boundary
    }
    return n as NonNegativeInt;
  },
};

function setRetries(n: NonNegativeInt): void {
  // No check needed. The type *is* the proof.
}
```

The throw still exists — but it lives in the parser (the boundary), not in every consumer.

## Pressure Resistance

### "Throwing is faster than constructing a `Result` object"

Both happen on input you don't control. The cost difference is nanoseconds in code paths that, by definition, are not hot. The cost of `try` / `catch` sprawl in every caller is measured in code review hours.

### "Returning `undefined` means callers won't handle the missing case"

The compiler will refuse to let them ignore it. `T | undefined` shows up as a red squiggle on every `.field` access. That's the *enforcement*. Throws have no compile-time signal.

### "Throwing is more idiomatic in JavaScript"

Modern TS-with-Result patterns (Effect, Neverthrow, `safeParse`, tRPC, React Query) all return tagged results. The "idiom" is mid-evolution. Lead, don't follow.

### "It's defensive — what if I forget a check?"

That's the rule. By defining the error out of existence, *there is no check to forget*. `substring` that clamps cannot be called wrong. `getRole` that returns `T | undefined` cannot be called without handling absence.

### "Some errors really are exceptional — I can't return them"

Yes. Those are the ones to throw. The test: "Could a reasonable caller produce this input?" If yes, return. If no (network down, disk full, programming bug), throw — that's fail-fast territory.

### "My function does I/O — it has to be able to fail"

Two failure modes, two treatments. Validation failures (user input, schema mismatch) → return tagged result. Infrastructure failures (DB unreachable, timeout, OOM) → throw. The point of this rule is to stop conflating them.

## Red Flags

- A `throw` followed by a `try { ... } catch` at the only call site.
- A function with both a "validates" comment and a "throws if invalid" comment.
- A test that asserts `expect(() => fn(...)).toThrow(...)` for an input that came from a form.
- A `result.success` check followed by `if (!result.success) throw new Error(result.error)` — converting a tagged result back into an exception.
- A docstring listing more exception types than return types.
- A wrapper utility named `tryX` or `safeX` that exists *only* to make the throwing API usable.

**All of these mean: the API signature throws for normal input — reshape to return.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Throwing is shorter to write" | And longer to call. Net cost is negative. |
| "Result types don't compose well" | They compose via `await` + early-return on `!result.ok`. Same shape as `try` / `catch`, less syntax. |
| "Frameworks expect thrown errors" | At the *outermost* boundary, convert. Throw to the framework's error boundary if needed. Inside your own code, return results. |
| "Stack traces are better for debugging" | Tagged results carry the cause. Stack traces are noise when the "error" is "user typed a bad email." |
| "I'd have to refactor every caller" | Many callers were going to handle the case anyway — better to typecheck the handling than scatter try/catch. |
| "What if the result type explodes — `Result<T, A \| B \| C \| D>`" | That's a signal that the function is doing too many things. Split it, or fold related errors into a discriminated union. |

## Related

- `fail-fast` — the reconciled tension: expected input vs. invariants you control
- `errors-as-values` — handoff: failures that can't be defined away become typed values
- `parse-dont-validate` — narrow the input type so the bad case is unconstructable

## Reference

- John Ousterhout, *A Philosophy of Software Design* (2018), ch. 19 ("Define Errors Out of Existence") — the canonical chapter; the `substring` example is his.
- Joe Duffy, ["The Error Model"](http://joeduffyblog.com/2016/02/07/the-error-model/) (2016) — distinguishes recoverable returns from abnormal exceptions. The architectural argument for why most "errors" should be values.
