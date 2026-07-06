---
name: functional-core-imperative-shell
description: Use when a function under test requires mocking the DB, the clock, or a third-party API just to exercise its logic. Use when business logic and side effects are tangled together in one place. Use when a request handler that validates, computes, then writes can't have its computation tested without the surrounding I/O.
---

# Functional Core, Imperative Shell

## Overview

**Prefer pure functions for business decisions**—data in, data out, with no I/O, clock, randomness, or network. Keep orchestration and effects in an imperative shell.

The split is a design default, not a demand to extract every trivial transformation. Extract the decision-making core when logic and effects are tangled, independently useful, or hard to test through the public boundary.

## The Default Rule

```
Prefer a pure decision core and an effectful orchestration shell. Keep trivial glue together
when extraction would only add indirection; never hide I/O inside a function presented as pure.
```

Extract a core when business branches, calculations, time, or randomness need direct testing. Keep a thin pass-through in the shell when it contains no independent decision and an integration test exercises the behavior more clearly than another named function would.

## Why

Pure functions are the cheapest code to maintain in any codebase:
- Tested without mocks.
- Predictable — same input, same output, always.
- Reorderable, parallelizable, cacheable.
- Composable — outputs of one feed inputs of another.
- Trivial for an AI agent to refactor without breaking anything.

Side effects are the expensive part — they need network, clocks, fixtures, retries, and timeout handling. Concentrating them in a *thin shell* means the expensive code is small, named, and replaceable. The expensive code is also the only code that's hard to test, so it's the only code that needs test doubles.

The wrong shape is logic-with-effects sprinkled through ten functions. The right shape is **lots of pure logic, a little I/O around the edge.**

## Detection

You are violating the rule if any of these are true:

- A function described as "calculates X" also reads from the DB or calls an API.
- A test for business logic mocks `Date`, `Math.random`, `fetch`, or the database.
- A pure-looking helper (`formatInvoice`, `computeTax`) silently reads env or calls `Date.now()`.
- A function takes no inputs and returns a value — it's reading hidden state.
- A function returns the same shape but two calls give different results.
- The same business calculation is duplicated across a route, a job, and a test because none of the three could call the others (each was tangled with its own I/O).

## The Pattern

### Handler — shell calls core

```ts
// ❌ Logic and I/O tangled. Untestable without mocking three things.
async function applyDiscountAndCharge(orderId: OrderId, code: string) {
  const order = await orders.findById(orderId);
  const discount = await discounts.findByCode(code);
  if (!discount || discount.expiresAt < new Date()) {
    throw new Error('expired');
  }
  const total = Math.round(order.totalCents * (1 - discount.percent / 100));
  await payments.charge({ amountCents: total, customerId: order.customerId });
  await orders.markPaid(orderId, new Date());
}

// ✅ Core: pure, testable, named.
type DiscountInput = {
  orderTotalCents: number;
  discountPercent: number;
  discountExpiresAt: Date;
  now: Date;
};
type DiscountResult =
  | { ok: true; chargeCents: number }
  | { ok: false; reason: 'expired' };

export function computeDiscountedCharge(input: DiscountInput): DiscountResult {
  if (input.discountExpiresAt < input.now) return { ok: false, reason: 'expired' };
  const cents = Math.round(input.orderTotalCents * (1 - input.discountPercent / 100));
  return { ok: true, chargeCents: cents };
}

// ✅ Shell: thin, imperative, only-effects.
async function applyDiscountAndCharge(orderId: OrderId, code: string) {
  const order = await orders.findById(orderId);
  const discount = await discounts.findByCode(code);
  if (!discount) throw new Error('expired');

  const result = computeDiscountedCharge({
    orderTotalCents: order.totalCents,
    discountPercent: discount.percent,
    discountExpiresAt: discount.expiresAt,
    now: new Date(),
  });

  if (!result.ok) throw new Error(result.reason);

  await payments.charge({ amountCents: result.chargeCents, customerId: order.customerId });
  await orders.markPaid(orderId, new Date());
}
```

The pure `computeDiscountedCharge` is tested with `it('returns expired when discount.expiresAt < now', ...)` — no mocks, no fixtures, no async. Three lines.

The shell is hard to test, but it's small and structurally obvious: load, compute, persist. If the shell breaks, you read three function calls top-to-bottom.

### Components — separate calculation from rendering

The same shape: data fetch (shell), pure functions compute (core), components render (shell).

```ts
// ✅ Pure — testable without a renderer.
export function pricingSummary(items: readonly LineItem[], taxRate: number) {
  const subtotal = items.reduce((sum, i) => sum + i.priceCents * i.qty, 0);
  const tax = Math.round(subtotal * taxRate);
  return { subtotal, tax, total: subtotal + tax };
}

// Shell — fetch + render.
function Cart({ items, taxRate }: { items: LineItem[]; taxRate: number }) {
  const summary = pricingSummary(items, taxRate);
  return <CartSummary {...summary} />;
}
```

### Pass the clock in

```ts
// ❌ Hidden clock — every test has to mock Date.
function isExpired(token: Token): boolean {
  return token.expiresAt < new Date();
}

// ✅ Explicit clock — test by passing different `now`.
function isExpired(token: Token, now: Date): boolean {
  return token.expiresAt < now;
}
```

Same for `Math.random`, `crypto.randomUUID`, file reads, env reads. They become *parameters* to the pure core, *invocations* in the shell.

Threading `now` as a parameter is the *purest* form, and it's the right call when a function's result genuinely turns on the clock (an expiry calculation, a scheduling decision). But it isn't the only honest option, and the library's default is the lighter one: for a function whose sole impurity *is* the clock, keeping `Date.now()`/`new Date()` inline and freezing the runner clock with fake timers in the test is idiomatic and reads cleaner at every call site — `mock-only-across-architectural-boundaries` and `seams-for-untestable-code` both take this position, and this skill defers to them for the clock specifically. Reach for a threaded (required, never defaulted) `now` when the clock is one of *several* inputs you're already passing, or when the pure function is reused across contexts that supply different times; a defaulted `now = new Date()` is the one form to avoid either way, because the signature *looks* injectable while still reading the clock at every call site that omits the arg.

### What counts as "I/O" — be honest

I/O is anything not deterministic from arguments:
- Reading/writing DB, files, network, queues
- Reading the current time (`Date.now()`, `new Date()`)
- Generating randomness (`Math.random()`, `crypto.randomUUID()`)
- Reading env vars (`process.env.X`)
- Reading mutable module state (singletons that change)

A function that does none of these can be tested without setup. That's the marker.

## Pressure Resistance

### "Passing `now` to every function is verbose"

Yes. It's also honest. The verbosity is a *signal* about how much code depends on the clock — usually less than you think. After a few weeks the verbosity disappears into "obviously how we do this."

### "Mocking is fine, tests work"

Tests that pass with mocks tell you the mocks work. They don't tell you the system works. Pure-function tests, by contrast, exercise actual logic with actual inputs. The difference is the difference between proof and theater.

### "Some things can't be pure — DB queries, network calls"

Right. Those live in the shell. The core is the *part that can be pure*. Most apps have far more pure logic than they realize; it's just buried inside I/O-tangled functions waiting to be extracted.

### "Splitting feels artificial — they're conceptually one thing"

They're conceptually one *feature*. They're structurally two concerns: *what we compute* and *what we do with the result*. The split is what makes each piece independently understandable.

### "I'll do this when the feature grows"

The feature grows by adding logic. The longer it stays tangled, the more logic accumulates inside the I/O wrapper, and the bigger the extraction job becomes. Easier early.

## Red Flags

- A "pure" function with `new Date()`, `Math.random()`, `await`, or any import from `db/`, `fetch`, `fs`, `process.env`.
- A test that needs the DB, the network, or module-mocking to reach a business *decision* — the decision is tangled with I/O and wants extracting. (Freezing the clock for a function whose *sole* impurity is the current time is fine — see "Pass the clock in".)
- A helper used by both a route and a job whose two callers each do "their own" pre-processing.
- A computation duplicated in three places because none of the existing copies could be reused.
- A function that returns `Promise<T>` but doesn't actually `await` anything — it's mid-extraction.
- A function with no parameters returning a value.

**All of these mean: I/O is buried inside logic — pull it out to the shell.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's faster to write inline" | Faster to write, slower to test, slower to reuse, slower to debug. Net cost negative. |
| "The function only needs the DB, why not call it directly" | Because then *every consumer* needs the DB. The pure version is callable by tests, by jobs, by the UI. |
| "Mocks let me test the same thing" | Mocks test *the interface to the mock*. Pure tests test *the logic*. Not equivalent. |
| "Splitting requires more files" | Two files of clear purpose beat one file of mixed purpose. |
| "Passing `now` everywhere is silly" | Less silly than three failed tests because someone's machine clock was wrong. |
| "The handler is the shell, so impurity is fine *anywhere* inside it" | Impurity is fine in the shell. The logic *called by* the shell should still be pure. |
| "We already inject the DB/repositories, so we follow the pattern" | Injecting infrastructure and passing the clock/randomness are *separate* obligations. DI removes the DB from your logic; it does nothing for an inline `new Date()`. Extract the time-dependent check into a pure function that takes `now`. |

## Related

- `test-observable-behavior-not-implementation` — a pure core means behavior tests need no mocks
- `mock-only-across-architectural-boundaries` — mocks are confined to the shell
- `immutability` — pure functions are naturally immutable

## Reference

- Gary Bernhardt, ["Boundaries"](https://www.destroyallsoftware.com/talks/boundaries) — Destroy All Software (2012). The original talk introducing *Functional Core, Imperative Shell* as a design pattern.
- Mark Seemann, ["Impureim sandwich"](https://blog.ploeh.dk/2020/03/02/impureim-sandwich/) (2020) — the same shape named from the FP side: impure–pure–impure.
- Scott Wlaschin, *Domain Modeling Made Functional* (2018) — a pure decision core with I/O pushed to the edges, worked through a full domain.
