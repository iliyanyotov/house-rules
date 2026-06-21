---
name: yagni
description: Use when designing a new feature, API, or abstraction. Use when tempted to add a config option "for flexibility," extract an abstraction "in case we have more," or build production-grade infrastructure for a prototype. Use when reviewing a PR that adds capabilities the requirement didn't ask for. Use when the words "might," "probably," "eventually," or "in case" appear in your justification.
---

# YAGNI (You Aren't Gonna Need It)

## Overview

**Build exactly what's required now.** No options for hypothetical future callers, no abstractions for the second use case that doesn't exist, no production hardening for a prototype.

The cheapest code is the code you don't write.

## When to Use

- Tempted to add a config option no caller currently sets
- Extracting an abstraction on the first occurrence
- Adding a feature flag "to be safe" for a change with no rollout risk
- Pre-building cache / retry / queue infrastructure before measuring need
- Adding schema fields the product doesn't use yet

## The Iron Rule

```
NEVER add capability before a real need exists. Wait for the second occurrence.
```

**No exceptions:**
- Not for "it's easy to add now"
- Not for "users will probably want it"
- Not for "best practices say to include it"
- Not for "I'm planning for scale"

## Detection: The Speculation Smell

If your justification contains "might," "probably," "in case," or "eventually" — STOP:

```typescript
// ❌ VIOLATION: premature plugin shape. One consumer, but already an
//    interface, a registry, and a dispatcher "in case we add SMS later."
interface Notifier {
  send(userId: UserId, body: string): Promise<void>;
}
const notifiers: Record<string, Notifier> = {
  email: new EmailNotifier(),
};
export function notify(channel: string, userId: UserId, body: string) {
  return notifiers[channel].send(userId, body);
}

// ✅ CORRECT: one consumer, one concrete function. Add the abstraction
//    when SMS actually arrives — and you'll know its real shape then.
export async function sendEmailNotification(userId: UserId, body: string): Promise<void> {
  await mailer.send(userId, body);
}
```

The "ready for SMS" version is *less ready*. When SMS arrives, the interface will be wrong (SMS has no subject line, has character limits, has different delivery semantics). You'll restructure anyway. Better to start concrete.

## Why YAGNI Wins

| Speculative code | YAGNI code |
|---|---|
| Bets you know the future | Solves the problem you have |
| Shapes the codebase around guesses | Leaves shape open for real requirements |
| Carries maintenance cost forever | Costs only when the need is real |
| Wrong-shape abstractions block real ones | Concrete code refactors cleanly |
| "Generic" tests must cover every branch | Tests only behavior that exists |

**Scope:** YAGNI applies to **capabilities**, **abstractions**, **configurability**, and **infrastructure beyond current stakes**. It does *not* apply to type safety, input validation at edges, error handling at boundaries, or tests for behavior that exists — those are baseline requirements, not speculative features.

## Pressure Resistance

### 1. "It's easy to add now, hard to add later"

**Pressure:** "Better to bake it in while I'm here."

**Response:** Almost always false. "Now" is when you have the least information. "Later" you'll know the real shape.

**Action:** Build the concrete version. Refactoring concrete-to-abstract is easier than abstract-to-concrete *because* you'll have the requirements then.

### 2. "Users will probably want this"

**Pressure:** "I can imagine them asking for it."

**Response:** "Probably" is not a requirement. Engineers' guesses about user needs are wrong most of the time.

**Action:** Wait for the second user request or measured signal. Until then, it's one engineer's guess.

### 3. "Best practices say to include this"

**Pressure:** "The textbook says production systems need pagination / rate-limit / audit log."

**Response:** Best practices are answers to *problems*. Don't apply the answer if you don't have the problem.

**Action:** Match infrastructure to actual stakes. The right practice for 1M req/s is different from 100 req/day.

### 4. "It's the same effort to make it generic"

**Pressure:** "Specific or generic, the code is the same size."

**Response:** Almost never true. Generic requires designing the abstraction, naming concepts, testing all branches, documenting the contract. Specific is a function.

**Action:** Write the specific version. The cost difference is 2–10× in code and 5–20× in maintenance.

### 5. "I'm adding it for future-me"

**Pressure:** "Future-me will thank me for this hook."

**Response:** Future-you has more information than current-you. Trust them to add the right thing then.

**Action:** Don't burden future-you with current-you's guess.

### 6. "We have to plan for scale"

**Pressure:** "We need the architecture to handle 100× growth."

**Response:** Plan, yes. *Build*, no. Premature scaling is the leading cause of premature complexity.

**Action:** Document the scaling path in an ADR. Build the simple version. Scale when measurement demands it.

## Red Flags

- The words "might," "probably," "eventually," "just in case," "to be safe" in a comment or PR description
- An interface with one implementation
- A `Factory`, `Strategy`, `Manager`, `Provider`, `Adapter`, `Wrapper` with one concrete behavior
- A configuration option with one value used in production
- A feature flag added without a documented rollout/rollback plan
- A schema column populated by no current code path
- A function parameter that callers always set the same way
- The phrase "while I'm at it, let me also…"
- An abstraction extracted on the first occurrence

**All of these mean: delete the speculation. Build the concrete version.**

## Quick Reference

| Symptom | Action |
|---|---|
| Interface with one implementation | Delete the interface; inline the class |
| Config option no caller sets | Delete the option; hardcode the default |
| `Strategy` / `Factory` / `Manager` with one mode | Inline as a function |
| "Just-in-case" schema column | Drop the column; add it when needed |
| Pre-built cache / retry without measurement | Remove; add when profiling shows the need |
| Feature flag with no rollout plan | Remove; ship the change directly |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|---|---|
| "It's hardly any extra code" | Until you sum across the codebase. Speculation compounds. |
| "Refactoring later is hard" | Refactoring concrete-to-abstract is easier than abstract-to-concrete. |
| "We need to be production-ready from day 1" | "Production-ready" depends on your production. Match the stakes. |
| "I'm just being thorough" | Thorough applies to the *current* problem, not imagined ones. |
| "It's a best practice in this domain" | Best practices solve specific problems. Apply when you have the problem. |
| "I'll forget if I don't add it now" | Write a TODO in the issue tracker, not in the codebase. |

## The Bottom Line

**Build what's asked. Skip what's imagined. Refactor when reality arrives.**

Speculative code is a bet on the future. The base rate of those bets paying off is low — and the wrong-shape abstraction is more expensive than the missing one.

## Related

- `kiss` — sibling simplicity guards
- `dry` — premature extraction is the "abstraction for maybes" YAGNI prohibits

## Reference

- Ron Jeffries & Kent Beck, *Extreme Programming Explained* (1999) — *"Always implement things when you actually need them, never when you just foresee that you need them."*
- Donald Knuth — *"Premature optimization is the root of all evil."* (Same principle, performance flavor.)
- Martin Fowler, "Yagni" (2015) — the canonical modern essay on the rule.
