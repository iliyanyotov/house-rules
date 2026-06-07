---
name: law-of-demeter
description: Use when typing `a.b.c.d` to reach a value. Use when a function parameter is a wide object but only one nested field gets used. Use when a unit test fixture has 5 levels of nested objects mostly so the call chain in the system under test works. Use when a refactor that renames `b` requires touching everywhere that reads `a.b.c.d`. Use when "I just need to dig into this object" feels routine.
---

# Law of Demeter

## Overview

**Talk only to your immediate neighbors.** A function should access its own parameters, its own local variables, and the methods/fields of those — **one level deep**. If you find yourself writing `a.b.c.d`, ask the *caller* to give you `c` directly, or ask `a` for the operation you actually want.

The point isn't dot-counting. The point is that `a.b.c.d` *encodes knowledge* about `b`'s and `c`'s internal structure — knowledge that should be inside `a` or `b`, not at the call site.

## The Iron Rule

```
NEVER drill more than one level into a parameter. Take the value you need, or ask the immediate neighbor.
```

**No exceptions:**
- Not for "it's just navigation, what's the harm?"
- Not for "I only do this in one place"
- Not for "the chain reads naturally as English"
- Not for "optional chaining handles nulls — same thing"

## Why

Chained navigation produces *implicit coupling*: a function that reads `order.customer.billingAddress.country` is coupled not just to `Order`, but to `Customer`'s structure, to `Address`'s structure, and to the fact that customers have billing addresses, separately from shipping. Rename `billingAddress` to `billing`, and every site doing this chain breaks.

The fix is to *delegate* — let the immediate neighbor answer the question. Either:

1. **The caller passes what's needed.** Instead of `function foo(order: Order)` digging into `order.customer.billingAddress.country`, write `function foo(country: Country)` and let the caller resolve the path.
2. **The module exposes the operation.** Add `getBillingCountry(order)` to the `order` module so the chain lives in one place.

The wins:

1. **Refactors localize.** Renaming `billingAddress` → `billing` requires editing one accessor, not 50 call sites.
2. **Tests are simpler.** A function that takes `country: Country` is tested with a country, not a full order with nested customer and nested address.
3. **Coupling is visible in signatures.** `function foo(order: Order)` looks low-coupling — until you read the body. `function foo(country: Country, ...)` is honest about what it actually needs.
4. **Mental load drops.** Reading `a.b.c.d` requires holding the entire path's type in your head; reading `country` doesn't.

## Detection

You are violating the rule if any of these are true:

- A chain of 3+ field accesses or method calls in one expression: `a.b.c.d` or `a.foo().bar().baz()`.
- A unit test must construct a deeply nested fixture mostly to support a chain in the system under test.
- A rename of an intermediate field in a chain breaks many call sites.
- A function takes a wide object (`Order`, `User`, `Request`) but only one or two deeply-nested fields are used.
- A `?.?.?.?` chain because each level might be null — and you handle the null at the end rather than at the source.
- A code comment "navigates the X graph" or "drills into Y" near the chain.
- Optional chaining (`?.`) appears 3+ times in one expression.

## The Pattern

### Pass the value, not the container

```ts
// ❌ The function reaches three levels in.
function calculateShipping(order: Order): Money {
  const country = order.customer.billingAddress.country;
  return shippingTable[country] ?? Money.zero('USD');
}
```

```ts
// ✅ The caller resolves the path; the function takes the value.
function calculateShipping(country: Country): Money {
  return shippingTable[country] ?? Money.zero('USD');
}

// Caller (which already has the order — and the path lives in one place):
const shipping = calculateShipping(order.customer.billingAddress.country);
```

The chain still exists, but at *one* call site — the caller. The pure function doesn't know about orders, customers, or addresses; it knows about countries and shipping. Refactoring the order/customer/address shape doesn't touch `calculateShipping`.

### Ask the immediate neighbor for the answer

When the chain is *common* enough that callers shouldn't reproduce it:

```ts
// ❌ Multiple consumers all dig into `order` to get the country.
const country1 = order1.customer.billingAddress.country;
const country2 = order2.customer.billingAddress.country;
const country3 = order3.customer.billingAddress.country;
```

```ts
// ✅ The order module exposes the operation.
export function getOrderBillingCountry(order: Order): Country {
  return order.customer.billingAddress.country;
}

const country1 = getOrderBillingCountry(order1);
const country2 = getOrderBillingCountry(order2);
```

Now the chain lives in one place. If `billingAddress` becomes `billing`, one function changes; all callers stay valid.

### Don't dig — message the immediate neighbor

The OO formulation is *"don't talk to strangers."* In functional TS, the equivalent: a function may freely access its own parameters and their *one-level* members. Reaching deeper means you're talking to a stranger's friend's neighbor.

```ts
// ❌ The function asks its parameter for its friend's status.
function isOrderShipped(order: Order): boolean {
  return order.shipment.tracking.events.some((e) => e.kind === 'delivered');
}
```

```ts
// ✅ The order module delegates to the shipment module.
export function isOrderShipped(order: Order): boolean {
  return isShipped(order.shipment);
}

export function isShipped(shipment: Shipment): boolean {
  return hasDeliveredEvent(shipment.tracking);
}

export function hasDeliveredEvent(tracking: Tracking): boolean {
  return tracking.events.some((e) => e.kind === 'delivered');
}
```

Each module handles its own structure. Renaming `events` to `eventLog` requires touching the tracking module only.

### One-dot-per-line as a soft heuristic

The folk version of Demeter: *"only one dot per line of code."* It's a heuristic, not a rule — `array.map(...)` is two dots and obviously fine. But when you see 3+ dots between *named entities* (not method-chained calls), that's the signal.

```ts
// ✅ Fine — method chaining on the same entity (array operations).
items.filter((x) => x.active).map((x) => x.name).join(', ');

// ❌ Suspicious — chain across named entities.
result.user.account.permissions.flags.includes('admin');
```

The distinction: the first chain operates on a *single value* through transformations; the second drills through *multiple entities*. Drilling is the Demeter violation.

### When to ignore the rule

Some chains are unavoidable and fine:

- **Standard library / framework APIs.** `req.headers.get('authorization')` is two dots; the framework chose that shape.
- **JSON-like data structures.** When parsing external data with a schema, the parsed object has the shape the API gave you. Don't artificially flatten.
- **Pure transformations.** `array.map().filter().sort()` is fluent and idiomatic.

The rule applies to *your own domain types* whose structure *you control*. External APIs and built-in primitives are exempt.

## Pressure Resistance

### "It's just navigation, what's the harm?"

The harm is encoding `b.c.d`'s structure at every site that does the navigation. When `c`'s structure changes, every site changes. With a delegating function, one site changes.

### "Adding accessor functions is bloat"

The accessor is one line. The 30 inline chains across the codebase are 30 places that have to stay in sync. Net code shrinks.

### "Counting dots is silly"

The rule isn't about dots — it's about *crossing module boundaries* in a single expression. Dot-count is a useful proxy because crossing usually involves a dot. Don't fixate on the count; fixate on the boundary crossings.

### "I need the value — what choice do I have?"

You have three: (1) accept it directly as a parameter; (2) ask the immediate neighbor for an operation that gives you what you need; (3) accept that the rule has an exception for external APIs whose structure you don't own.

### "Optional chaining handles nulls — same thing"

`a?.b?.c?.d` is *worse* — it papers over the fact that each step could be missing without naming why. The fix isn't more `?.`; it's eliminating the chain.

### "Refactoring all the call sites is too much work"

Apply incrementally — when you'd write `a.b.c.d`, write `getXFromA(a)` instead and put the chain in `a`'s module. Over time, the codebase converges. Don't refactor everything at once.

## Red Flags

- Three or more dots between named entities in a single expression.
- A `?.?.?.?` chain with multiple optional steps — each one is a place that might fail.
- A test fixture file with 5+ levels of nested objects mostly to support production chains.
- A rename PR that touches 30 files because they all read `a.b.c.d`.
- A function parameter typed as a wide object (`Order`, `User`) whose body uses only a deeply-nested subset.
- A comment "navigates X graph" or "drills into Y."

**All of these mean: the function is reaching across module boundaries — delegate or pass the value directly.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's clearer to see the path inline" | Until the path changes. Then it's the loudest noise in the codebase. |
| "Accessor functions add indirection" | One named indirection beats many inline navigations. |
| "I only do this in one place" | Until tomorrow when you copy-paste it to a second place. |
| "Demeter is OO theory" | Demeter is about *cohesion and coupling* — applies to any module system. |
| "TypeScript ensures the path exists" | TS ensures it *types* — it doesn't ensure it stays the same shape across refactors. |
| "The chain reads naturally as English" | English is forgiving; refactors aren't. |

## Reference

- Karl Lieberherr and Ian Holland, ["Assuring good style for object-oriented programs"](https://www2.ccs.neu.edu/research/demeter/papers/law-of-demeter/oopsla88-law-of-demeter.pdf) (1989) — the original formulation. Named after the Demeter Project at Northeastern University.
- Andy Hunt & Dave Thomas, *The Pragmatic Programmer* (1999), §10 ("It's Just a View") — the modern programmer-facing restatement: *"reduce coupling by writing 'shy' code."*
