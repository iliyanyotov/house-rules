---
name: kiss
description: Use when tempted to write clever code. Use when the solution feels complex for what it accomplishes. Use when reaching for a type-level construct, a deep method chain, or a "smart" one-liner. Use when a junior engineer would have to study the code to read it.
---

# KISS (Keep It Simple, Stupid)

## Overview

**The simplest implementation that solves the actual problem wins.**

Code is read far more often than it's written. Cleverness is a cost paid by every future reader, including future-you at 3am.

## When to Use

- Choosing between a direct approach and a clever one
- Tempted to use a deeply chained expression or a one-liner
- Reaching for type-level computation when a hand-written type would do
- Writing code that requires re-parsing to understand
- A reviewer's first question would be "what does this do?"

## The Iron Rule

```
NEVER choose clever over clear. Simple wins.
```

**No exceptions:**
- Not for "it's more elegant"
- Not for "it's a nice one-liner"
- Not for "it's technically faster"
- Not for "this is how experts write it"

## Detection: The "Clever" Smell

If you're proud of how clever your code is, simplify it:

```typescript
// ❌ VIOLATION: clever one-liner. Three transformations, opaque comparator.
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

## Why Simple Wins

| Clever code | Simple code |
|---|---|
| Compresses logic into dense syntax | Spreads logic across readable steps |
| Demands re-parsing on every read | Scans on the first read |
| Failure modes hidden in nested calls | Failure modes visible in plain branches |
| Hostile error messages from type magic | Hand-readable errors from hand-written types |
| Survives only as long as its author | Survives team turnover |

## Pressure Resistance Protocol

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

### 5. "It's a fun puzzle to read"

**Pressure:** "Reading clever code is part of the craft."

**Response:** Code isn't a puzzle. Code is documentation that runs.

**Action:** Rewrite. If a colleague needs to puzzle out your code during an incident, you owe them an apology.

## Red Flags — STOP and Reconsider

- A line that takes longer to read than to copy-paste
- Nested ternaries (`x ? a : y ? b : c`)
- A method chain longer than 4 calls without an intermediate variable
- A regex longer than ~30 characters without a comment
- Type-level wizardry (`infer`, conditional types, deep mapped types) where a plain type would do
- A function whose generic signature is more code than its body
- The PR description contains "elegant," "clever," "neat trick," or "one-liner"

**All of these mean: rewrite simply.**

## Quick Reference

| Symptom | Action |
|---|---|
| "Elegant" one-liner | Expand to clear multi-line |
| Nested ternary | Convert to `if` / `else` |
| Method chain ≥ 5 calls | Break into named intermediate steps |
| Long inline regex | Several simple checks, or a commented constant |
| Type-level inference | Hand-write the types |
| Magic numbers | Named constants |
| Pride in cleverness | Rewrite simply |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|---|---|
| "It's a one-liner, that's the point" | One-liners that take 30s to read are compressed, not concise. |
| "Functional style is cleaner" | Sometimes. When it's not, switch styles in that function. |
| "Smart types are self-documenting" | Only to whoever wrote them. To everyone else, they're opaque. |
| "It's a small project, who cares" | Small projects grow. The reader six months from now has no context. |
| "We need to look modern" | The most sophisticated code looks boring. Boring enables fast change. |

## The Bottom Line

**Simple beats clever. Clear beats concise. Obvious beats elegant.**

Find the simplest implementation that works. If a tired engineer can't read it at midnight, simplify it.

## Reference

- Kelly Johnson, Lockheed Skunk Works (1960) — *"Keep it simple, stupid."* The original engineering version: simple systems are repairable in the field.
- John Ousterhout, *A Philosophy of Software Design*, ch. 21 — "Different ≠ Better." Avoid complexity for novelty's sake.
- Antoine de Saint-Exupéry — *"Perfection is achieved not when there is nothing more to add, but when there is nothing left to take away."*
- Donald Knuth — *"Premature optimization is the root of all evil."*
