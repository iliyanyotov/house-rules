---
name: fail-fast
description: Use when designing a function that operates on inputs from another module. Use when tempted to write defensive `if (x == null) return` early-returns that silently substitute defaults. Use when an invariant is violated and the choice is between throwing now or "hoping it works out." Use when reviewing code that catches an error and falls through with no log, no re-throw, and a default value.
---

# Fail Fast

## Overview

**Detect invariant violations at the earliest possible moment and stop.** Throw at the boundary where the violation is first observable. Never substitute a "safe default" for invalid input and continue.

A function that accepts garbage and produces a value is worse than one that crashes — the crash is loud; the bad value is silent.

## The Iron Rule

```
NEVER swallow an invariant violation. Throw at the boundary where it's first observable.
```

**No exceptions:**
- Not for "the user will get a 500"
- Not for "it might be transient"
- Not for "graceful degradation"
- Not for "I'll log it and continue"

## Why

Bugs caught at the boundary point to their cause. The stack trace names the violator. The fix is localized.

Bugs that propagate as silent default values surface three layers later, in a feature that has nothing to do with the cause. Hours of debugging trace the wrong thread. By the time someone finds the real source, the original context is gone.

This rule applies to invariants you *control* — env vars, function preconditions, internal contracts, programmer assumptions. It does not apply to input from outside (user forms, third-party responses); that input is *expected* to be malformed and gets a structured error response, not an exception. The distinction is the boundary.

## Detection

You are violating the rule if any of these are true:

- A function body starts with `if (!x) return null` for an `x` whose type says it's non-null.
- A `try/catch` block catches an error and falls through with a default value — no log, no re-throw. (The *logged* variant counts too: a `log.warn` plus a sentinel return is still the caller proceeding on missing data, just with a paper trail.)
- A `switch` over a discriminant has `default: return defaultValue` over a closed set.
- A function uses `?.` chains to "protect against" properties the type system says exist.
- An env-var read defaults to a placeholder (`process.env.KEY ?? 'sk_test_dummy'`) without verifying the env in development.
- A function named `safe*`, `try*`, or `maybe*` wraps a call only to swallow its failure.

## The Pattern

### Module-load-time invariants — check at the boundary, not per-call

```ts
// ❌ Checked on every request. Stays silent if the env never had the key.
export async function chargeCustomer(amountCents: number) {
  const key = process.env.PAYMENT_SECRET_KEY;
  if (!key) {
    console.error('payment key missing'); // who reads this?
    return null;
  }
  // ...
}

// ✅ Fail at module load. The process refuses to start without the key.
import { z } from 'zod';

const env = z.object({
  PAYMENT_SECRET_KEY: z.string().startsWith('sk_'),
  DATABASE_URL: z.string().url(),
}).parse(process.env);

export async function chargeCustomer(amountCents: number) {
  // `env.PAYMENT_SECRET_KEY` is `string`. No per-call check.
  await paymentClient(env.PAYMENT_SECRET_KEY).charge(amountCents);
}
```

If you reach for a per-key getter with a fallback instead of one upfront parse — `getEnv('PAYMENT_SECRET_KEY', '')` — the *fallback argument is the leak*. Passing `''` or a placeholder for a variable that should be required silently disables the throw: the process starts, and the failure resurfaces later as a confusing runtime error far from the cause. A fallback is legitimate only when the placeholder is genuinely valid in *every* environment; otherwise omit it and let the getter throw.

The throw happens once, at startup, with a clear message. Deployment fails loudly instead of every request silently returning `null` for hours.

### Function preconditions — assert what the type can't capture

```ts
// ❌ "Defensive" — but the default propagates the bug.
function chargeCents(amount: number): number {
  if (amount < 0) return 0;
  return Math.round(amount);
}

// ✅ The contract says positive. Throw on violation.
function chargeCents(amount: number): number {
  if (amount < 0) throw new RangeError(`chargeCents: amount must be >= 0, got ${amount}`);
  return Math.round(amount);
}

// ✅✅ Better — make the bad case unrepresentable.
function chargeCents(amount: NonNegativeInt): number {
  return Math.round(amount);
}
```

The last form pushes the throw to the boundary where `NonNegativeInt` is parsed. The interior trusts.

### Caught-and-ignored — the most common smell

```ts
// ❌ The bug is invisible. The caller sees a successful response with empty data.
async function loadOrders(userId: UserId): Promise<Order[]> {
  try {
    return await orders.findByUser(userId);
  } catch (err) {
    return []; // shrug
  }
}

// ✅ Let it propagate. The outermost boundary decides how to respond.
async function loadOrders(userId: UserId): Promise<Order[]> {
  return orders.findByUser(userId);
}

// In the handler — translate the failure at the *outer* boundary, with context.
export async function handleOrdersRequest(userId: UserId) {
  try {
    const data = await loadOrders(userId);
    return { status: 200, body: { data } };
  } catch (err) {
    log.error(err, { context: 'loadOrders', userId });
    return { status: 500, body: { error: 'load_failed' } };
  }
}
```

The error is handled in exactly one place — the outer boundary. Inside the system, code is short and assumes success.

A sneakier variant reassigns to a degraded-but-working alternate instead of returning a sentinel:

```ts
// ❌ Worse than `return []` — there's no empty value to betray the failure.
let key = await loadSigningKey();
try { key = await loadRotatedKey(); } catch { key = legacyKey; }  // silently runs on the wrong key
```

There's no empty result here to tip anyone off; the system just quietly runs on the wrong path. A `log.warn` in the catch does *not* redeem it — the caller still proceeds as if nothing failed. If the rotated key is required, let the failure propagate.

### The exception: input from outside

User input is *expected* to be invalid. That's not fail-fast territory — it's the parse-at-the-boundary discipline. Returning `{ ok: false, error: 'invalid_email' }` is correct because the caller is the UI, which knows how to display it.

Fail-fast applies to **invariants you control** — env vars, function preconditions, internal state. Not to user-supplied data, which has its own discipline.

## Pressure Resistance

### "But the user will get a 500 if I throw"

Yes — and a 500 with a stack trace in your error tracker is recoverable. A 200 with empty data is *not* recoverable, because the user assumes the system worked. Failure modes the operator can see beat failure modes only the user can see.

### "It's resilient — the app should degrade gracefully"

Graceful degradation is a *designed* fallback at a known boundary (cache miss, third-party outage), with a logged event. It's not "swallow every error in every function." Resilience without observation is denial.

### "What if it's a transient error, like a network blip?"

Retry it. Explicitly. With backoff and a max attempt count. Not by `try/catch { return [] }`. Retry and swallow are different operations.

### "I'll catch and log so we know about it"

Catch and re-throw with context is fine: `throw new Error('loadOrders failed', { cause: err })`. Catch-and-log-and-continue is the smell — the continuation hides the bug from the caller, who now operates on bogus state.

### "Throwing in async functions creates ugly stack traces"

Use `cause`. The chain is preserved across the await. Node and most error trackers render it.

## Red Flags

- A `try/catch` whose catch body has no `throw` and no log.
- A `?.` chain on a value whose type says it's defined.
- A function that returns `null` or `[]` from inside a catch.
- A catch that logs *and* returns a sentinel (`[]`, `null`, `''`) with a comment claiming "graceful degradation" — the log and comment make it *look* deliberate, but the caller still proceeds on missing data. A real graceful-degradation fallback is a designed boundary (see Pressure Resistance), not a leaf return.
- An env-var fallback to a hardcoded placeholder (`'localhost'`, `'sk_test_dummy'`).
- A test that asserts a function "doesn't throw" for clearly invalid input — that test enshrines the bug.
- The phrases "just in case," "for safety," or "defensive" without a documented failure mode.
- A `default: return ...` in a switch over a closed discriminant — should be `assertNever`.

**All of these mean: the throw is missing, and the bug is now silent.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Throwing is too aggressive" | Aggression is silent corruption shipped to users. Throwing is honesty. |
| "It might be transient" | Retry explicitly. Don't mix up retry with swallow. |
| "I'll come back and fix the error handling" | The catch-and-default stays for the life of the codebase. |
| "Production should never crash" | Production should never *corrupt*. Crashing is a recoverable state; corrupting is not. |
| "Logging is enough" | Logging without re-throwing means the caller proceeded with broken state. The log is a tombstone, not a guard. |
| "The error message isn't actionable" | Then write a better error message. That's the work. |

## Related

- `define-errors-out-of-existence` — opposite boundaries: invariants-you-control vs. expected-external-input
- `swallow-deliberately-at-the-boundary` — the one permitted exception to "never hide an error"
- `errors-as-values` — throw on bugs; return typed values on expected failures

## Reference

- Jim Shore, ["Fail Fast"](https://martinfowler.com/ieeeSoftware/failFast.pdf) (IEEE Software, 2004) — the canonical statement: *"The sooner a problem is noticed, the cheaper it is to fix."*
- Michael Nygard, *Release It!* (2nd ed., 2018) — the production-grade case for fail-fast as the foundation of stability patterns.
