---
name: kiss
description: Use when explicitly reviewing code for unnecessary complexity. Use when reaching for a type-level construct, a deep method chain, or a "smart" one-liner where a plain approach would do. Use when a reviewer asks "why is this so complex?" and a straightforward version would make the intent obvious.
---

# KISS (keep it simple, stupid)

## Overview

**The simplest implementation that solves the actual problem wins.**

Code is read far more often than it's written. Cleverness is a cost paid by every future reader, including future-you at 3am.

## When to use

- Choosing between a direct approach and a clever one
- Tempted to use a deeply chained expression or a one-liner
- Reaching for type-level computation when a hand-written type would do
- Writing code that requires re-parsing to understand
- A reviewer's first question would be "what does this do?"

## The heuristic

```
Treat "clever over clear" as a signal to simplify, not a law to cite. When a
line makes you pause to decode it, that pause is the smell — spend the effort
on clarity, not on the cleverness.
```

The posture is strong — simple wins, and elegance is never a reason to ship something harder to read — but "clever" isn't a bright line, so this is a diagnostic, not an invariant. The concrete smells to act on: a method chain past ~5 calls, a regex past a screenful, type-level machinery where a plain type would do, a one-liner that needs a comment to parse.

**What does *not* trip it:**
- A short, idiomatic chain (`items.filter(...).map(...)`) — declarative and clear is *simple*, not clever.
- A genuinely-needed abstraction that removes more complexity than it adds.
- Brevity that is *also* clearer than the long form. The rule is against clever-*instead-of*-clear, not against concise.

## Detection: the "clever" smell

If you're proud of how clever your code is, simplify it:

```typescript
// ❌ VIOLATION: clever one-liner. Two chained transformations, destructured parameters, an inline clock read.
const overdueIds = invoices
  .filter(({ dueAt, paidAt }) => paidAt == null && dueAt.getTime() < Date.now())
  .map(({ id }) => id);

// ✅ CORRECT: same result, less to parse.
const now = new Date();
const overdueIds: InvoiceId[] = [];
for (const invoice of invoices) {
  if (invoice.paidAt == null && invoice.dueAt < now) {
    overdueIds.push(invoice.id);
  }
}
```

The chain isn't *wrong* — it's the *default reach*. Boring is the baseline; cleverness is an optimization that has to earn its place.

## Why simple wins

| Clever code | Simple code |
|---|---|
| Compresses logic into dense syntax | Spreads logic across readable steps |
| Demands re-parsing on every read | Scans on the first read |
| Failure modes hidden in nested calls | Failure modes visible in plain branches |
| Hostile error messages from type magic | Hand-readable errors from hand-written types |
| Survives only as long as its author | Survives team turnover |

## Pressure resistance

### 1. "It's more elegant"

**Pressure:** "This one-liner is more elegant than the verbose version."

**Response:** Elegance is clarity, not brevity. A line that takes 30 seconds to read costs more than three lines that take 5.

**Action:** Use the clear version. Name intermediate variables.

### 2. "It shows advanced skills"

**Pressure:** "I want to demonstrate I can use the advanced features."

**Response:** Senior engineers are recognized for solutions a junior can maintain. Cleverness is a debt the team pays.

**Action:** Solve the problem boringly. Save cleverness for the few places it's truly required.

### 3. "It's technically faster"

**Pressure:** "The complex version avoids an extra allocation."

**Response:** Premature optimization. Is this actually a measured bottleneck, or is it a guess?

**Action:** Write the simple version. Optimize only when profiling shows a real hot path.

### 4. "Type-level magic prevents bugs"

**Pressure:** "This conditional type ensures the compiler catches a whole class of mistakes."

**Response:** Some do. Most produce error messages no one can decode and bugs no one would have written anyway.

**Action:** Write the type by hand. If it doesn't catch a real bug a unit test would miss, it's not earning its complexity.

*Exception:* a conditional type earns its keep when it derives a type that genuinely *can't* be hand-written — e.g. a polymorphic `as`-prop component whose accepted props must be `JSX.IntrinsicElements[T]`. The test is whether a plain type expresses the same constraint, not whether `infer` appears. (A conditional type that merely reshapes a known query result is the kind to delete.)

### 5. "It's a fun puzzle to read"

**Pressure:** "Reading clever code is part of the craft."

**Response:** Code isn't a puzzle. Code is documentation that runs.

**Action:** Rewrite. If a colleague needs to puzzle out your code during an incident, you owe them an apology.

## Red flags

- A line that takes longer to read than to copy-paste
- Nested ternaries over un-named conditions (`x ? a : y ? b : c`) — the offense is the inlined logic, not the ternary form; name the conditions or extract a lookup
- A method chain longer than 4 calls without an intermediate variable — *or* any chain (any length) that mixes transforms with a `.sort()`/in-place mutation, or chains off a possibly-`undefined` value
- A regex longer than ~30 characters without a comment
- Type-level wizardry (`infer`, conditional types, deep mapped types) where a plain type would do
- A function whose generic signature is more code than its body
- The PR description contains "elegant", "clever", "neat trick", or "one-liner"

**All of these mean: rewrite simply.**

## Quick reference

| Symptom | Action |
|---|---|
| "Elegant" one-liner | Expand to clear multi-line |
| Nested ternary as a *statement* (branching control flow) | Convert to `if` / `else` |
| Nested ternary assigning one value to a `const` | Extract a helper or lookup map that returns the value — keep it `const`, don't introduce a mutable `let` |
| Method chain ≥ 5 calls | Break into named intermediate steps |
| Long inline regex | Several simple checks, or a commented constant |
| Type-level inference | Hand-write the types |
| Magic numbers | Named constants |
| Pride in cleverness | Rewrite simply |

## Common rationalizations (all invalid)

| Excuse | Reality |
|---|---|
| "It's a one-liner, that's the point" | One-liners that take 30s to read are compressed, not concise. |
| "Functional style is cleaner" | Sometimes. When it's not, switch styles in that function. |
| "Smart types are self-documenting" | Only to whoever wrote them. To everyone else, they're opaque. |
| "It's a small project, who cares" | Small projects grow. The reader six months from now has no context. |
| "We need to look modern" | The most sophisticated code looks boring. Boring enables fast change. |

## The bottom line

**Simple beats clever. Clear beats concise. Obvious beats elegant.**

Find the simplest implementation that works. If a tired engineer can't read it at midnight, simplify it.

## Related

- `yagni` — sibling simplicity guards (clever syntax vs. speculative scope)
- `dry` — neither abstracts for elegance alone

## Reference

- Kelly Johnson, Lockheed Skunk Works (1960) — *"Keep it simple, stupid."* The original engineering version: simple systems are repairable in the field.
- John Ousterhout, *A Philosophy of Software Design*, ch. 17 ("Consistency") — a "better idea" is not sufficient excuse to deviate; avoid complexity for novelty's sake.
- Antoine de Saint-Exupéry — *"Perfection is achieved not when there is nothing more to add, but when there is nothing left to take away."*
- Donald Knuth — *"Premature optimization is the root of all evil."*
