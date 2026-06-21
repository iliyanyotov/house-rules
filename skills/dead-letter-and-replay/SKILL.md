---
name: dead-letter-and-replay
description: Use when consuming an at-least-once event stream you don't control — provider webhooks (payments, auth, messaging), a message queue, a pub/sub topic. Use when a handler that throws causes the event to be lost, or to be retried forever by the producer. Use when "we missed a webhook and the data is now out of sync" or "a poison message is jamming the queue" appears in an incident. Use when a consumer has retries but no place for the events that exhaust them.
---

# Dead-Letter and Replay

## Overview

**An event consumer that exhausts its retries must persist the raw event to a dead-letter store rather than drop it — and there must be a safe, idempotent way to replay it later.** Inbound events you don't control are not recoverable by re-asking; if you lose one, it's gone. The dead-letter store is the safety net under every consumer.

This pairs with two sibling skills: dedupe each delivery on arrival (`idempotency-keys-on-writes`, consumer side), and when the failure is a *downstream* dependency being down, fail fast on it (`circuit-breaker-on-flaky-deps`) — then dead-letter what you couldn't process.

## The Iron Rule

```
NEVER let a failed inbound event vanish or retry unbounded. Persist the raw event, cap retries, make replay idempotent.
```

**No exceptions:**
- Not "the producer retries, so we don't need to store it"
- Not "it'll basically never fail"
- Not "we'll just re-sync everything on a cron" (that's a different, lossy guess)
- Not "we log the error, that's enough" (a log line is not a replayable event)

## Why

You control your own writes; you can re-run them. You do **not** control a third party's event stream. When a webhook handler throws, one of two bad things happens: the producer gives up after its own retry budget and the event is **lost forever**, or the producer retries indefinitely and a single **poison event** hammers you on every redelivery. Either way, your data silently drifts out of sync with the source of truth, and you find out days later from a confused user.

The fix is structural. Store the **raw, unparsed** event the moment processing fails, with enough metadata to replay it. Bound in-line retries so a poison event can't loop. Make the handler idempotent so replay — manual or automated — is always safe. The dead-letter table becomes the durable record of "things we received but couldn't yet process," and replay turns a lost event into a recoverable one.

## Detection

You are violating the rule if any of these are true:

- A webhook/queue handler that throws has no path that persists the event.
- A consumer relies entirely on the producer's retries with no local durable copy.
- A poison message can be redelivered forever with no cap and no quarantine.
- The recovery story for a missed event is "re-sync the whole entity from the API."
- A handler catches, logs the error, and returns 200 — the event is acknowledged *and* lost.
- There's a dead-letter table but no tested way to replay from it.
- Replaying an event would double-apply its side effects (no idempotency).

## The Pattern

### Sub-pattern 1 — Dedupe on arrival, then process

Before doing work, claim the event by its provider-supplied ID in the same transaction as the side effect. This is the consumer side of `idempotency-keys-on-writes` and it's the precondition for safe replay.

```ts
async function handleEvent(raw: string, signature: string) {
  const event = verifyAndParse(raw, signature);          // authenticate first

  await db.transaction(async (tx) => {
    const claimed = await tx.processedEvents.insertOnce({ eventId: event.id });
    if (!claimed) return;                                 // duplicate delivery — no-op
    await apply(tx, event);                               // the real work
  });
}
```

### Sub-pattern 2 — On failure, dead-letter the RAW event

If processing throws, persist the original payload — unparsed — plus what you need to retry it and debug it. Store raw, because the failure may *be* a parse/schema mismatch; a parsed copy would lose the very bytes you need.

```ts
async function handleEvent(raw: string, signature: string) {
  try {
    const event = verifyAndParse(raw, signature);
    await processIdempotently(event);
  } catch (err) {
    await deadLetters.create({
      raw,                       // exact original bytes — replay needs these
      signature,                 // so replay can re-verify
      source: 'payment-provider',
      error: serializeError(err),
      attempts: 1,
      receivedAt: new Date(),
    });
    reportError(err, { context: 'event-dead-lettered', source: 'payment-provider' });
    // ACK the producer (return 2xx) once it's safely stored — you own recovery now.
  }
}
```

Note the acknowledgement decision: once the event is durably in your dead-letter store, return success to the producer. You have taken ownership of recovery; letting the producer keep retrying a poison event buys nothing.

### Sub-pattern 3 — Bound in-line retries before dead-lettering

A transient blip deserves a couple of immediate retries; a poison event deserves none. Cap attempts, then dead-letter. (Retries use backoff + jitter and only wrap idempotent work — see `retry-with-jitter-and-budget`.)

```ts
async function handleWithBudget(raw: string, signature: string, maxAttempts = 3) {
  for (let attempt = 1; ; attempt++) {
    try {
      return await processIdempotently(verifyAndParse(raw, signature));
    } catch (err) {
      if (attempt >= maxAttempts || !isTransient(err)) {
        await deadLetters.create({ raw, signature, error: serializeError(err), attempts: attempt, receivedAt: new Date() });
        return;
      }
      await sleep(backoffWithJitter(attempt));
    }
  }
}
```

### Sub-pattern 4 — Replay must be idempotent and safe

Replay is just "run the handler again on the stored bytes." Because the handler dedupes on arrival (sub-pattern 1) and the work is idempotent, replaying is safe whether the original partially applied or not at all. Delete (or mark resolved) on success.

```ts
async function replayDeadLetter(id: DeadLetterId) {
  const dl = await deadLetters.findById(id);
  if (!dl) return;
  await handleEvent(dl.raw, dl.signature);   // same path — re-verifies, re-dedupes, re-applies
  await deadLetters.markResolved(id);        // or delete
}
```

Expose replay as an operator action (admin endpoint, CLI) and/or an automated sweep that retries dead letters with a capped, backed-off schedule. **Test the replay path** — an untested dead-letter table is a graveyard, not a safety net.

### Sub-pattern 5 — Purge resolved entries

The dead-letter store grows; left alone it grows forever. Delete on successful replay, and run a retention sweep for resolved/expired rows. (This is `steady-state-purge-unbounded-growth` applied to the safety net.)

## Pressure Resistance

**"The provider already retries — storing the event is redundant."** The provider retries on *its* schedule and gives up on *its* budget, neither of which you control or can see. When it gives up, the event is gone for good. And a poison event the provider keeps redelivering is a denial-of-service on your handler until you can quarantine it. Your durable copy is the only recovery mechanism you own.

**"We'll just re-sync the whole entity from the API if we miss something."** Re-syncing is a *guess* that you noticed the gap, that the API still exposes the lost transition, and that a full refetch reconstructs it. Many events are deltas (`order.refunded`, `subscription.canceled`) whose effect can't be reconstructed from current state. Dead-lettering preserves the actual event; re-sync hopes you can reverse-engineer it.

**"Logging the error is enough — we'll see it and fix it."** A log line is not replayable. You can read it, but you can't re-drive the side effect from it, and you certainly can't re-verify the signature from a truncated log. Persist the structured, raw event so recovery is a button, not an archaeology project.

**"Catch, log, return 200 keeps the producer happy."** It does — and it discards the event in the same breath. Acknowledging *and* dropping is the worst outcome: the producer thinks you handled it, so it never retries, and you have nothing. ACK only after the event is durably stored.

**"Adding a dead-letter table and replay is a lot of machinery."** It's one table (raw payload, source, error, attempts, timestamps) and one function that re-invokes the existing handler. The idempotency you need for replay you already need for at-least-once delivery. The marginal cost is small; the cost of a silently lost financial event is not.

## Red Flags

- A webhook/queue handler whose `catch` neither persists the event nor re-throws to a framework that will.
- A consumer with no durable record of events that failed processing.
- A redelivery loop with no attempt cap and no quarantine.
- "Re-sync from the API" as the entire missed-event recovery plan.
- A handler that returns 2xx from inside a catch without storing anything.
- A dead-letter table with no replay function, or a replay function with no test.
- Replay that would double-charge / double-create because the handler isn't idempotent.
- A dead-letter table that has never been purged and nobody knows its size.

**All of these mean: the event can be lost or double-applied — persist it raw, cap retries, make replay idempotent, and test it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The producer's retries cover us" | Their budget and schedule, not yours; when they quit, it's gone. |
| "It almost never fails" | The one time it does is a financial event drifting out of sync silently. |
| "We'll re-sync from the source" | Deltas can't be reconstructed from current state; re-sync is a hopeful guess. |
| "The log has the error" | A log isn't replayable and often isn't the raw bytes. Store the event. |
| "Return 200 so it doesn't retry" | ACK-and-drop is the worst case — handled in name, lost in fact. |
| "Dead-letter + replay is over-engineering" | One table and one re-invoke. You already need the idempotency. |
| "We have a dead-letter table" (but never replay from it) | An untested replay path is a graveyard. Exercise it. |

## Reference

- AWS, [*Amazon SQS dead-letter queues*](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) — the canonical dead-letter mechanism: isolate messages that can't be processed so they neither block the queue nor disappear.
- Enterprise Integration Patterns, [*Dead Letter Channel*](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DeadLetterChannel.html) (Hohpe & Woolf) — the foundational pattern for undeliverable messages.
- Extends `idempotency-keys-on-writes` (dedupe each delivery so replay is safe); pairs with `retry-with-jitter-and-budget` (bound in-line attempts) and `circuit-breaker-on-flaky-deps` (when the failure is a downstream outage).
