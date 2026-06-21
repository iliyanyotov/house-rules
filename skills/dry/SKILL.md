---
name: dry
description: Use when reviewing code with apparent duplication. Use when tempted to extract a helper to avoid repeating two or three lines. Use when designing a shared utility module. Use when reviewing a "DRY refactor" PR that introduces parameters or branches to unify previously-separate code paths. Use when the words "since we're duplicating this anyway" come up.
---

# DRY — Deduplicate *Knowledge*, Not Lines

## Overview

**DRY deduplicates a single piece of *knowledge*, not a single piece of *text*.** If two code paths happen to look alike but represent different decisions, leave them alone. Three coincidentally similar lines is better than a premature abstraction.

The signal that demands DRY is *"if this rule changes, we'd need to update both places for the same reason."* Without that signal, looking-alike is not duplication.

## The Iron Rule

```
NEVER deduplicate code that only looks alike. Deduplicate when the same knowledge is repeated.
```

**No exceptions:**
- Not for "there's duplication, we should DRY it up"
- Not for "three lines × three places is clearly DRY territory"
- Not for "what if we forget to update one"
- Not for "the abstraction makes intent clearer"

## Why

The original DRY principle (Hunt & Thomas, 1999) is *"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."* The keyword is *knowledge* — a business rule, a domain concept, a configuration value.

The common misreading is "any two identical lines must be extracted." That misreading creates a worse problem than duplication: **wrong coupling**. Two code paths that look alike now but represent different concerns will sooner or later need to diverge. If they were prematurely unified:

1. The "shared" function grows parameters and branches until it serves both concerns awkwardly — coupled, slower to change, harder to read.
2. Someone changes the shared function "for case A," accidentally breaks case B — silent, painful, hard to detect.

Sandi Metz's operative phrase: **"Duplication is far cheaper than the wrong abstraction."**

## Detection — Two Directions

You can violate the rule in either direction.

### Under-DRY (real knowledge duplicated)

- A business rule (tax rate, threshold, expiry window) is hardcoded in 3+ places.
- A validation schema is rewritten by hand in client and server.
- A constant (`30 * 24 * 60 * 60 * 1000`) appears in five files.
- A domain calculation lives next to two different consumers, slightly divergent.

### Over-DRY (text deduplicated, knowledge split)

- A shared helper has parameters that one caller always sets to `false` and another always sets to `true`.
- A "generic" function with a `mode` flag that branches into two unrelated implementations.
- A base class whose subclasses override most methods, sharing only setup boilerplate.
- A utility named `processItem` called from three different features doing fundamentally different things.

## The Pattern

### Real DRY — extract the knowledge

```ts
// ❌ Same rule, three places. Change one, miss the others.
// invoices/list.ts
if (invoice.dueAt < new Date() && !invoice.paidAt) { /* overdue */ }
// jobs/notify.ts
const overdue = invoices.filter(i => i.dueAt < new Date() && !i.paidAt);
// reports/totals.ts
const overdueTotal = invoices.reduce((sum, i) =>
  i.dueAt < new Date() && !i.paidAt ? sum + i.totalCents : sum, 0);

// ✅ One rule, one home. Tests live next to the definition.
// domain/invoices.ts
export function isInvoiceOverdue(invoice: Invoice, now: Date): boolean {
  return !invoice.paidAt && invoice.dueAt < now;
}

// Everywhere else:
const now = new Date();
if (isInvoiceOverdue(invoice, now)) { /* ... */ }
const overdue = invoices.filter(i => isInvoiceOverdue(i, now));
const overdueTotal = invoices
  .filter(i => isInvoiceOverdue(i, now))
  .reduce((sum, i) => sum + i.totalCents, 0);
```

This is *real* DRY. The knowledge — "what makes an invoice overdue" — has one authoritative source. Changing it (e.g., add a grace period) is one edit, propagated by the type system.

### Fake DRY — same text, different knowledge

The hardest case is code that's literally identical today but serves *different rules*. Same text ≠ same knowledge.

```ts
// pricing/loyalty.ts
// Loyalty members get 10% off their order total.
function applyLoyaltyDiscount(totalCents: number): number {
  return Math.round(totalCents * 0.9);
}

// pricing/clearance.ts
// Items on clearance are sold at 10% off the listed price.
function applyClearanceDiscount(priceCents: number): number {
  return Math.round(priceCents * 0.9);
}
```

The functions are character-identical. The temptation:

```ts
// ❌ "Same code, let's DRY it up."
function applyTenPercentOff(amountCents: number): number {
  return Math.round(amountCents * 0.9);
}
// pricing/loyalty.ts   → applyTenPercentOff(totalCents)
// pricing/clearance.ts → applyTenPercentOff(priceCents)
```

Three months later, the rules diverge:
- Loyalty changes to *tiered* discounts — 10% for silver, 15% for gold, 20% for platinum.
- Clearance stays flat 10% but adds a *minimum sale price* — clearance items can't go below $1.

Now `applyTenPercentOff` becomes:

```ts
// ❌ Same function, growing parameters as each caller adds its case.
function applyTenPercentOff(
  amountCents: number,
  opts?: { tier?: 'silver' | 'gold' | 'platinum'; minCents?: number },
): number { /* ... */ }
```

The function now serves two unrelated concerns through a single signature. A loyalty bug-fix risks breaking clearance pricing. Reviewers can't tell which `opts` field affects which call site. The "DRY" has become coupling.

```ts
// ✅ The right move was to leave them apart from the start.
//    The text was identical; the *rules* were always different.
function applyLoyaltyDiscount(totalCents: number, tier: LoyaltyTier): number {
  /* tier-specific logic, owned by loyalty domain */
}
function applyClearanceDiscount(priceCents: number): number {
  /* min-price logic, owned by clearance domain */
}
```

Each function now changes for one reason only.

The test: *if this rule changes, do **both** call sites need to update for the **same reason**?* For the loyalty/clearance case, no — they each change for their own reasons, on their own schedules, under their own stakeholders. They were never the same knowledge.

When in doubt, **leave the duplication and let divergence pressure tell you what to extract.** Wrong-abstraction is paid every commit; duplication is paid once, at extraction time, if at all.

### The "extract on third occurrence" heuristic

**Don't extract on the first occurrence. Consider on the second. Do it on the third — but only if all three are still the same kind of thing.**

The first occurrence has zero evidence of duplication. The second has anecdotal evidence. The third confirms a pattern *and* reveals what's actually shared (versus what just looks alike).

Not a hard rule — sometimes obvious knowledge (a tax rate constant) is extractable immediately. The point: *resist* premature DRY; *favor* duplication over wrong abstraction.

### Schema as a single source — DRY across runtime/compile-time

```ts
// ✅ One schema; types and runtime validation share a source.
export const Order = z.object({
  id: z.string(),
  totalCents: z.number().int().positive(),
  placedAt: z.coerce.date(),
});
export type Order = z.infer<typeof Order>;

// Server boundary parses with Order.parse(...)
// Client form validation uses Order.parse(...)
// Tests use Order.parse(...) for fixtures
```

This is real DRY. One schema, multiple consumers, one authoritative definition.

### Configuration — extract on first occurrence

Constants, magic numbers, env-derived values are *always* knowledge:

```ts
// ❌ Sprinkled.
setTimeout(fn, 30000);
expect(elapsed).toBeLessThan(30000);

// ✅ Named, single source.
const REQUEST_TIMEOUT_MS = 30_000;
setTimeout(fn, REQUEST_TIMEOUT_MS);
expect(elapsed).toBeLessThan(REQUEST_TIMEOUT_MS);
```

The knowledge — *our timeout is 30 seconds* — has one home.

## Pressure Resistance

### "There's duplication, we should DRY it up"

Maybe. Ask: *are these the same knowledge or do they just look alike?* If different concerns, leave them. If the same concern, extract.

### "Three lines duplicated three times — clearly DRY territory"

Three lines × three places isn't enough signal. Read what each does in its surrounding context. If they're doing the same *domain operation*, extract. If they're three coincidentally similar lines in three unrelated features, leave them.

### "But what if we forget to update one?"

That's a real risk for real knowledge duplication. It's not a risk for code-shaped duplication: if two unrelated paths happen to look alike, they *will* diverge, and the "forget to update one" concern wasn't a concern — the codepaths shouldn't have been updated in lockstep anyway.

### "The abstraction makes intent clearer"

Sometimes. A well-named function (`isInvoiceOverdue`) does. A `processData(items, options)` does not — it hides intent behind an options bag. Naming earns the abstraction; genericism doesn't.

### "I already extracted, now there's an awkward `if` branch"

You hit a wrong abstraction. The fix is to *un-extract* — inline the function back into both callers, then extract differently, or accept the duplication. Sandi Metz: *"Sometimes the right answer is to put back the duplication."*

## Red Flags

- A function with a boolean parameter that switches between two unrelated implementations.
- A "shared" utility with growing parameters as each new caller adds their special case.
- A base class whose subclasses override most methods.
- A generic helper named after a verb (`process`, `handle`) with no domain noun.
- Two callers that happen to call the same helper but for *different business reasons*.
- A "DRY refactor" PR that produces deeper indentation, more parameters, or more conditional branches than the original.
- The phrase "since both X and Y do similar things" without examining *why* they're similar.

**All of these mean: the abstraction is wrong — either un-extract or accept the duplication.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Duplication is always bad" | Wrong abstraction is worse than duplication. |
| "I can always refactor later" | Easier to extract from duplicates than to un-extract from wrong abstractions. |
| "The DRY principle is universal" | The DRY *principle* is about knowledge; the *literal interpretation* is about text. They're different. |
| "This will cause inconsistency" | If the two codepaths represent the same rule, extract. If they don't, "consistency" was the wrong goal. |
| "The shared function is reusable" | Reusable for *what*? Specifically. If no second use case exists, reusability is hypothetical. |
| "We always extract helpers in this codebase" | Then your codebase carries the wrong-abstraction tax. Stop. |

## Related

- `kiss` — DRY names knowledge, not text; no abstraction for elegance
- `yagni` — don't extract for a maybe-future caller

## Reference

- Andy Hunt & Dave Thomas, *The Pragmatic Programmer* (1999) — the canonical statement: *"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."*
- Sandi Metz, ["The Wrong Abstraction"](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction) (2016) — *"Duplication is far cheaper than the wrong abstraction."* Required reading on this rule.
