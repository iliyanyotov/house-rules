---
name: comments-as-why-not-what
description: Use when writing a comment. Use when reviewing a PR where every block has a one-line preamble describing what the next lines do. Use when an existing comment "explains" what well-named code already expresses. Use when a comment references "the task" or "the ticket" — context that doesn't survive into the future. Use when a `// TODO` has been in the codebase >6 months. Use when JSDoc on a function repeats the types the signature already declares.
---

# Comments Explain Why, Not What

## Overview

**A comment exists to say something the code *cannot*** — a non-obvious constraint, a hidden invariant, a surprising trade-off, a workaround for a specific external bug, a reference to an authoritative source. If removing the comment would not confuse a future reader who understands the language, don't write the comment.

The bar: *would I tell someone reading this for the first time in person?* If yes, comment it. If you'd say "the code already says that" — delete it.

## The Iron Rule

```
NEVER write a comment that paraphrases the next line of code.
```

Code already explains *what* — that's what names, types, and short functions are for. Comments explain *why*: what the code can't.

## The Pattern: Categories of Legitimate Comments

### 1. Delete what-comments — every time

```ts
// ❌ All three comments paraphrase the next line.
function processOrder(order: Order) {
  // Validate the order
  validateOrder(order);
  // Charge the customer
  chargeCustomer(order.customerId, order.totalCents);
  // Send confirmation email
  sendConfirmationEmail(order);
}

// ✅ Same code; the function names are the comments.
function processOrder(order: Order) {
  validateOrder(order);
  chargeCustomer(order.customerId, order.totalCents);
  sendConfirmationEmail(order);
}
```

If the next reader doesn't understand `chargeCustomer` from its name, the name is wrong — fix the name; don't paper over it with a comment.

### 2. Keep why-comments — the non-obvious

```ts
// ✅ Hidden constraint not visible in the code.
// Payment provider charges fail silently for amounts below 50 cents.
// We throw here so failures surface at validation, not at charge time.
if (amountCents < 50) {
  throw new Error('amount_below_provider_minimum');
}

// ✅ Workaround for a specific external bug.
// The provider's webhook returns 200 with body { error } on partial failure
// (not 4xx). Detect by body shape until they fix it — tracked in their #4421.
if ('error' in webhookResponse) throw new Error(webhookResponse.error);

// ✅ Surprising trade-off the next reader might "improve."
// Re-running the parse here is intentional — the value flows through a
// serialization boundary that strips the branded type. Re-parse restores it.
const restored = User.parse(payload);
```

In each case the code *itself* doesn't and can't say what the comment says. Delete the comment and the next reader is one bug or one wasted hour away from rediscovering the constraint the hard way.

### 3. JSDoc — only when it adds something the signature doesn't

```ts
// ❌ Redundant JSDoc — restates the types.
/**
 * Fetches a user by ID.
 * @param id - The user ID
 * @returns The user
 */
async function fetchUser(id: UserId): Promise<User> { /* ... */ }

// ✅ JSDoc earns its place by adding non-obvious information.
/**
 * Returns `undefined` for soft-deleted users (vs throwing).
 * Soft-deleted rows have `deletedAt IS NOT NULL` and are excluded by the
 * default scope. Pass `{ includeDeleted: true }` for admin audit views.
 */
async function fetchUser(
  id: UserId,
  opts?: { includeDeleted?: boolean },
): Promise<User | undefined> { /* ... */ }
```

The first is noise. The second names two non-obvious things (soft-delete behavior, when to use the option) — information the signature alone doesn't carry.

### 4. `// TODO` — owner + issue, or delete

```ts
// ❌ Orphan TODO — accumulates forever.
// TODO: handle the retry case better

// ✅ Owner + tracking issue + acceptance criterion.
// TODO(@alice, #1842): retry budget needs to be per-attempt, not total —
// currently a single slow attempt starves subsequent ones.
```

A TODO without an owner and an issue is going to live forever. If you can't write the owner and issue, the work isn't real — delete the TODO and accept the current behavior, or *do* the work now.

### 5. `// HACK` / `// WORKAROUND` — name the cause and the trigger to remove

```ts
// ✅ Names the underlying issue and the condition for removal.
// HACK: the runtime's `cacheTag` type doesn't include the readonly variant
// we get from `as const`. Spreading into a mutable array fixes the type.
// Remove when upstream ships #nnnn (tracked, open as of 2026-05).
revalidateTag([...readonlyTags]);
```

A bare `// HACK` is just a warning sign; one with context is *information* — the next person sees the issue is tracked and knows what fixes it.

### 6. Block comments — for high-level intent

```ts
// ✅ Names a multi-step design decision the code alone can't carry.
//
// Reconciliation strategy: write-then-confirm against the payment provider.
//   1) Insert the local order row with status='pending_charge'.
//   2) Call the provider's idempotent /charges endpoint.
//   3) On success → 'paid'. On retryable failure → keep 'pending_charge'
//      for the next reconciliation cron. On permanent failure → 'failed'.
//
// We treat the provider as the source of truth for "did the charge happen?"
// Local 'paid' is only set when we have a provider charge_id. A crash
// between (1) and (2) leaves an order in 'pending_charge' that the cron
// picks up — safe by design.
async function placeOrder(/* ... */) { /* ... */ }
```

This is the *good* kind of block comment. It explains a non-trivial *design decision* the code alone can't communicate — and which the next reader might otherwise "improve" by making (2) atomic with (1), breaking the recovery property.

### 7. Module-level orientation — when the file has quirks

```ts
// ✅ Module-level doc that orients the reader to non-obvious context.
//
// External feed ingestion service.
//
// The feed arrives daily as HTML. Two formats exist: pre-2020 (flat table)
// and 2020+ (nested tables with merged cells). The parser dispatches on
// the year and runs the matching variant.
//
// Known quirks:
//   - Some pre-2020 feeds use a non-breaking space as the thousands
//     separator. The number parser strips both regular and NBSP.
//   - One historical feed (2024-08) had a typo in a category name; the
//     parser accepts both the typo and the corrected spelling.
```

Information the reader cannot reasonably derive from the code, but absolutely needs before changing it.

## The Deletion Test

When uncertain about a comment, write it — then ask *"could I delete this and would a future reader be confused?"*

```ts
// ❌ Deletes cleanly — the code says the same thing.
// Increment the counter on each tick
counter++;

// ✅ Deletes badly — the comment preserves a hidden invariant.
// We count *attempts*, not successes — the rate limiter resets the
// counter on success.
counter++;
```

If removing the comment loses information, keep it. If it doesn't, delete it.

## Pressure Resistance

**"Comments help juniors."** Juniors read the code *and* the comment. If the comment paraphrases the code, they read the same information twice. If the comment carries something the code doesn't, they get context they couldn't have otherwise. The rule isn't "no comments for juniors"; it's "comments that *add* information, regardless of audience."

**"Comments help the AI assistant."** Same answer. A comment that paraphrases the code provides the assistant zero additional signal. A comment that names a hidden constraint helps the assistant exactly the same way it helps a human.

**"We always comment everything in our codebase."** Conventions are revisable. New code can establish a new convention; old comments get cleaned up opportunistically when the surrounding code is touched.

**"The linter requires JSDoc."** Configure the linter to require *meaningful* JSDoc only on *exported* APIs, not on every internal function. Most lint configs support this granularity.

**"I'm documenting for my future self."** Future-you reads code the same way present-you does. If present-you wouldn't need the comment, future-you won't either.

**"Comments narrate the change for the reviewer."** That's the PR description's job. The PR description is read once, by the reviewer. A comment in the code is read forever, by every future reader.

## Common Mistakes

| Smell | Fix |
|---|---|
| Comment paraphrases the next line of code | Delete; if the code is unclear, rename the function/variable instead |
| JSDoc restates parameter and return types | Delete; the signature already says it |
| `// Step 1` / `// Step 2` numbering | The function is too long; split it |
| `// TODO` older than 6 months | Add owner + issue link, or delete |
| `// HACK` with no description | Add the cause and the condition for removal, or fix it |
| Comment references "the task" or "this PR" | Delete; that context belongs in the PR description, not the code |
| Header comment with author + date | Delete; version control carries that |
| Multi-paragraph history of how the code came to be | Delete; `git blame` carries it for free |
| `// kept for backward compatibility` with no current callers | Delete the code and the comment |

## The Bottom Line

**Code says *what*. Comments say *why*.**

The deletion test is the single most useful filter: if removing the comment wouldn't confuse a future reader, the comment isn't earning its space. Keep the ones that preserve information the code can't.

## Reference

- Robert C. Martin, *Clean Code* (2008), ch. 4 — the canonical chapter on "Good Comments and Bad Comments." The good-comment list maps almost exactly onto "why."
- Steve McConnell, *Code Complete* 2e (2004), ch. 32 ("Self-Documenting Code") — *"The best documentation is the code itself."*
- John Ousterhout, *A Philosophy of Software Design* (2018), chs. 12–16 — the contrary positive case: writes *more* comments, but only ones that capture *intent* the code can't. Same rule, framed positively.
