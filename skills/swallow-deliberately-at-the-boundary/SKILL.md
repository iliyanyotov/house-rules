---
name: swallow-deliberately-at-the-boundary
description: Use when a side effect is optional and its failure must not fail the operation around it — a courtesy email after a committed order, an analytics event, a cache warm, a webhook notification. Use when tempted to write a bare `catch {}` or `catch { return }`. Use when an outer operation has already committed and a follow-on best-effort step throws. Use when `fail-fast` and "this must not block the main flow" seem to conflict. Use when reviewing a `catch` that has no `throw` and no log.
---

# Swallow Deliberately at the Boundary

## Overview

**An error may be swallowed only when the failing work is genuinely optional, the critical work has already committed, and the swallow is observable.** This is the disciplined inverse of `fail-fast`: that skill says *never* hide an error; this skill defines the one narrow seam where stopping an error is correct — and how to do it without it decaying into a silent `catch {}`.

`fail-fast` already names this exception ("a designed fallback at a known boundary, with a logged event"). This skill makes that exception a controlled, repeatable pattern instead of a loophole.

**"Optional" splits in two — and they get different handling:**
- **Fire-and-forget optional** — losing it costs nothing (an analytics event, a cache warm). Report and continue.
- **Deferred-but-owed optional** — it must not *block* the commit, but the user/system is genuinely owed it (a confirmation email, a receipt, a downstream sync). Report **and hand it to a retry path** (an outbox row or a background job) so it's *retried, not dropped*. A swallow that silently discards owed work is still a bug.

Swallowing-and-dropping is correct only for the first kind. For the second, swallow-and-defer.

## When to Use

- A best-effort side effect runs *after* the main transaction commits (confirmation email, receipt, push notification) — these are usually *owed*, so report **and** defer to a retry path.
- A telemetry/analytics call that must never affect the user-facing result — usually fire-and-forget, so report and drop.
- A cache warm, prefetch, or non-authoritative write whose failure is recoverable later.
- Any place you're about to type `catch {}` "so it doesn't blow up the request."

**Do NOT use** when the failing work is part of the operation's contract, when it runs *before* the critical write, or for input validation / invariant violations — those are `fail-fast` territory.

## The Iron Rule

```
NEVER swallow an error silently. Swallow only optional, post-commit work — and always record it.
```

**No exceptions:**
- A `catch` that swallows MUST report (error tracker and/or log) — no bare `catch {}`.
- The swallowed work MUST be optional — its failure leaves the system correct.
- The critical work MUST have already succeeded — you are not masking a failure of the real operation.
- If any of those three is false, this is not a swallow; it's `fail-fast`, and you must re-throw.

## Why

Some failures *should* stop everything (a charge that didn't go through). Some should not (the "your order shipped" email bounced). Treating the second kind like the first makes a committed, correct order report as a failed request — the user retries, you double-charge. Treating the first kind like the second corrupts state silently.

The danger is that the *correct* tool for the second case — a `catch` that doesn't re-throw — is identical in shape to the *worst* anti-pattern in the first case. The difference is invisible at a glance. So the discipline is: make every legitimate swallow loudly observable and provably optional, so a reviewer can tell it apart from a bug, and so the swallowed failure is never truly lost.

## Detection: The "silent catch" smell

```ts
// ❌ Indistinguishable from a bug. Is this deliberate? Is the work optional?
//    Did the real operation commit? Nobody can tell. The failure vanishes.
async function placeOrder(input: OrderInput) {
  const order = await orders.create(input);
  try {
    await mailer.sendConfirmation(order);
  } catch {}                                   // silent — the smell
  return order;
}

// ❌ Same smell, promise-chain form — the one people forget to look for.
await mailer.sendConfirmation(order).catch(() => {});

// ✅ Deliberate: the chain form reports too.
await mailer.sendConfirmation(order).catch((err) =>
  logger.error('sendConfirmation failed', { err, orderId: order.id }),
);

// ✅ Deliberate, observable, provably post-commit and optional.
async function placeOrder(input: OrderInput) {
  const order = await orders.create(input);    // critical work commits FIRST

  // Best-effort: a failed confirmation email must not fail a placed order.
  // We want to know it happened, but not retry the order over it.
  try {
    await mailer.sendConfirmation(order);
  } catch (err) {
    reportError(err, { context: 'sendConfirmation', orderId: order.id });
  }

  return order;
}
```

`reportError` here is shorthand for "make it observable" — a structured log line is an equally valid first option: `logger.error('sendConfirmation failed', { err, orderId: order.id })`. Use whichever your stack already routes and alerts on; the rule is that the catch is *not silent*, not that it calls a specific tracker.

The fix isn't "add a log line." It's the whole posture: critical work first, the optional work clearly labelled, the catch reporting with context, and a comment stating *why* this failure is tolerable.

### Report-and-drop vs. report-and-defer

The example above *reports* the email failure but still drops the email — fine if the email is a courtesy nobody is owed. But a confirmation email usually *is* owed: the user should get it, it just mustn't fail the order. For owed work, don't swallow it into a report and walk away — hand it to a retry path:

```ts
async function placeOrder(input: OrderInput) {
  const order = await orders.create(input);              // critical work commits FIRST

  // Owed, but must not block the order: enqueue it. The job retries with backoff;
  // if it ultimately exhausts, it dead-letters. The order is never held hostage to it.
  try {
    await jobs.enqueue('sendOrderConfirmation', { orderId: order.id });
  } catch (err) {
    // Even enqueue can fail — report; an outbox row written in the order's own
    // transaction is the sturdier variant when the email is truly must-deliver.
    reportError(err, { context: 'enqueue:sendOrderConfirmation', orderId: order.id });
  }

  return order;
}
```

Rule of thumb: **fire-and-forget → report-and-drop; owed → report-and-defer** (job/outbox, which then retries and, on exhaustion, dead-letters — see `dead-letter-and-replay`). Either way the catch is observable and the critical path is untouched; the difference is whether the optional work is allowed to vanish.

## Why Deliberate Swallowing Beats Both Extremes

| Approach | Result |
|---|---|
| Re-throw everything (naive fail-fast) | A bounced courtesy email fails a committed order; user retries; double side effects. |
| Swallow everything silently (`catch {}`) | Real failures vanish; corrupt state ships; debugging chases ghosts. |
| **Swallow deliberately at the boundary** | Optional failures are tolerated *and recorded*; critical failures still propagate. |

## Pressure Resistance

**Pressure: "Re-throwing here would fail the whole request over a non-critical email."**
Response: Correct — which is exactly why you swallow *this* call. But only after the order committed, and only with a `reportError`. The rule isn't "never swallow"; it's "never swallow silently, and never swallow the critical path."
Action: Confirm the critical write is above the `try`; add the error report; comment the why.

**Pressure: "Adding error reporting to every best-effort call is noise."**
Response: The report is the entire justification for the swallow. Without it you haven't degraded gracefully — you've gone blind. A swallow without observation is denial, not resilience.
Action: Always `reportError`/log in the catch. If that feels like too much per site, extract a `bestEffort(fn, context)` helper that does it for you.

**Pressure: "`fail-fast` says never swallow — so this skill contradicts it."**
Response: It doesn't. `fail-fast` governs invariants and the critical path; it explicitly permits designed, logged degradation at a boundary. This skill *is* that permitted seam, made rigorous. They compose: fail-fast for the operation, deliberate-swallow for its optional follow-ons.
Action: Ask "is this work part of the contract, or a courtesy after it?" Contract → fail-fast. Courtesy → swallow deliberately.

**Pressure: "I reported the failure, so swallowing the confirmation email is handled."**
Response: Reporting makes the failure *visible*; it doesn't make the user *whole*. If the work is owed (a receipt, a confirmation, a sync), a report-and-drop means the user simply never gets it — you just have a log proving you knew. Owed work goes to a retry path (job/outbox), which retries and, on exhaustion, dead-letters.
Action: Classify the optional work — fire-and-forget (drop) or owed (defer). For owed, enqueue it; don't let a `reportError` be the end of the story.

**Pressure: "The email failure might mean something is actually broken."**
Response: Then route it to your error tracker (which you're doing) and alert/monitor on it — that's how you find systemic problems. But don't punish the user whose order succeeded by failing their request. Observation and propagation are different responses; pick observation here.
Action: Report with enough context to investigate later; keep the user-facing result successful.

**Pressure: "It runs before the main write, but I'll swallow it anyway to be safe."**
Response: Then a failure of the *real* operation can hide behind the swallow, or the optional step's failure leaves the main step acting on bad assumptions. Order matters: optional, swallowed work goes *after* the commit.
Action: Reorder so the critical write is first and fully committed before any best-effort step.

## Red Flags

- A `catch` with an empty body, or `catch { return }` / `catch { return null }` with no report — including the promise-chain form `.catch(() => {})` / `.catch(() => null)`.
- A swallow positioned *before* the operation's critical write.
- Swallowing a call whose result the caller actually depends on.
- "It's fine if this fails" asserted in a comment but with no error report to prove it was considered.
- A swallowed `catch` around a whole multi-step block, so you can't tell *which* step was optional.
- Reaching for a swallow to silence a failure you don't understand yet.

**All of these mean: either re-throw (it's the critical path — `fail-fast`) or fix the swallow (narrow it, move it post-commit, add the report).**

## Quick Reference

| Situation | Action |
|---|---|
| Confirmation email / receipt after committed order | **Owed** → report **and defer** (job/outbox); don't drop |
| Downstream sync after a committed write | **Owed** → report and defer to a retry path |
| Analytics / telemetry event | Fire-and-forget → report and drop |
| Cache warm / prefetch | Fire-and-forget → report and drop; recompute on miss |
| The payment charge itself | Re-throw — `fail-fast` |
| Validation of incoming input | Structured error response — parse at boundary |
| Step the next step depends on | Re-throw — not optional |
| A whole block where only one call is optional | Narrow the `try` to the one optional call |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|---|---|
| "Empty catch keeps the request from failing" | It also keeps you from ever knowing it failed. Report, don't vanish. |
| "Logging every best-effort failure is too noisy" | The log is the license to swallow. No log, no swallow — re-throw instead. |
| "It's basically the same as graceful degradation" | Only if it's observable and post-commit. Otherwise it's a hidden bug. |
| "I'll wrap the whole flow in one try/catch" | Then you can't tell critical from optional. One swallow per optional call. |
| "Swallowing is fine, the user doesn't care about the email" | The user cares that their order succeeded — which is *why* you swallow the email, loudly, not the order. |
| "This failure is probably transient" | Transient → retry explicitly with backoff. Retry and swallow are different operations. |

## The Bottom Line

There is exactly one place a `catch` may decline to re-throw: an optional step, after the real work has committed, with the failure recorded. Everywhere else, `fail-fast`. Make every legitimate swallow loud enough that a reviewer can tell it from a bug at a glance.

## Related

- `fail-fast` — the inverse it carves an exception to
- `graceful-degradation-defaults` — sibling deliberate-fallback pattern
- `dead-letter-and-replay` — owed optional work: defer instead of drop

## Reference

- Michael Nygard, *Release It!* (2nd ed., 2018) — degradation as a *designed*, observed fallback at a boundary, not an ambient suppression of errors.
- Complements `fail-fast`: that skill forbids hiding invariant violations and the critical path; this one defines and disciplines the single boundary exception it allows.
