---
name: outbox-for-atomic-write-and-publish
description: Use when a handler commits a database change and then publishes an event, enqueues a job, or fires a webhook as a second, separate step. Use when consumers miss events whose business writes clearly committed, or receive events for writes that rolled back. Use when a `publish` call sits inside a `try/catch` that logs and continues, or inside a `db.transaction` callback. Use when an incident says "the order exists but the fulfillment service never heard about it".
---

# Outbox for Atomic Write-and-Publish

## Overview

**A state change and the event announcing it must commit atomically — so the event is written to an outbox table in the same database transaction as the business write, and a separate relay publishes it and marks it sent.**

Two operations against two systems (your database, your broker or webhook target) cannot both-succeed-or-both-fail on their own. The outbox turns "write + publish" into one transactional write plus asynchronous, retryable delivery.

## The Iron Rule

```
NEVER commit a state change and publish its event as two separate operations.
The event commits in the same transaction as the write; a relay owns delivery.
```

**No exceptions:**
- Not for "the publish almost never fails"
- Not for "we wrapped the publish in try/catch and log failures" (a log line is not delivery)
- Not for "we run a nightly reconciliation sync" (a lossy guess, hours late)
- Not for "the broker is highly available" (your process crashing between the two calls is the failure, not the broker)

Scope: the rule governs fire-and-forget *announcements* of a committed state change — domain events, queue jobs, outbound webhooks. A synchronous call whose response the handler needs before it can proceed (charging a card before confirming an order) is not an event and cannot be outboxed; that call needs its own idempotent-outbound discipline instead.

## Why

The dual write has two failure modes, and both are silent:

- **Lost event.** The transaction commits, then the process dies (deploy, crash, OOM) or the broker call fails before the publish completes. The invoice is `paid` in your database; no consumer ever learns. Data drifts, and you find out from a confused user days later.
- **Phantom event.** The publish happens inside (or before) the transaction, and the transaction then rolls back — a serialization failure, a constraint violation, a crash before commit. Consumers heard about a payment that never happened, and there is no unsend.

No ordering fixes this. Publish-after-commit loses events; publish-before-commit fabricates them; a `try/catch` just picks which one you get. The fix is structural: make the event *part of the transaction* by writing it as a row, then hand delivery to a relay that retries until the broker confirms.

## Detection

You are violating the rule if any of these are true:

- A handler awaits a DB write, then awaits `publish` / `enqueue` / a webhook `fetch` as a separate statement.
- A `publish` call appears *inside* a `db.transaction` callback — a network side effect that can't roll back with the transaction.
- A `catch` around a publish logs and returns success — the write committed, the event evaporated.
- Consumers are reconciled by a periodic "re-sync everything" cron instead of by events.
- A bug report says the entity changed but the downstream system "missed the webhook", and the producer has no durable record of what it owed.
- Two systems disagree about state and neither side can say which events were actually sent.

## The Pattern

### Write the event in the same transaction

```ts
export const outbox = pgTable('outbox', {
  id: uuid('id').primaryKey().defaultRandom(),
  topic: text('topic').notNull(),
  payload: jsonb('payload').$type<Record<string, unknown>>().notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  publishedAt: timestamp('published_at', { withTimezone: true }),
});
```

```ts
// ❌ Dual write. The crash window between the two awaits loses the event forever.
async function markInvoicePaid(invoiceId: InvoiceId, paymentId: PaymentId): Promise<void> {
  await db.update(invoices)
    .set({ status: 'paid', paidAt: new Date() })
    .where(eq(invoices.id, invoiceId));
  await publishEvent('invoice.paid', { invoiceId, paymentId }); // ← process dies here: committed, unannounced
}

// ✅ One transaction. The write and its announcement commit or roll back together.
async function markInvoicePaid(invoiceId: InvoiceId, paymentId: PaymentId): Promise<void> {
  await db.transaction(async (tx) => {
    await tx.update(invoices)
      .set({ status: 'paid', paidAt: new Date() })
      .where(eq(invoices.id, invoiceId));
    await tx.insert(outbox).values({
      topic: 'invoice.paid',
      payload: { invoiceId, paymentId, occurredAt: new Date().toISOString() },
    });
  });
}
```

The inverse mistake — moving the `publishEvent` network call *into* the transaction callback — is the phantom-event variant: the broker accepts the message, the commit then fails, and consumers act on a rollback. Only the outbox *insert* belongs inside; the network never does.

### The relay — claim, publish, mark sent

A background loop (worker, cron tick) drains unpublished rows. `FOR UPDATE SKIP LOCKED` lets multiple relay instances run without double-claiming a row that another instance is mid-publishing:

```ts
async function relayOutboxBatch(batchSize = 100): Promise<number> {
  return db.transaction(async (tx) => {
    const pending = await tx.select()
      .from(outbox)
      .where(isNull(outbox.publishedAt))
      .orderBy(asc(outbox.createdAt))
      .limit(batchSize)
      .for('update', { skipLocked: true });

    for (const row of pending) {
      await publishEvent(row.topic, { eventId: row.id, ...row.payload }, {
        signal: AbortSignal.timeout(5_000),
      });
      await tx.update(outbox).set({ publishedAt: new Date() }).where(eq(outbox.id, row.id));
    }

    counter('outbox.relayed', pending.length, {});
    return pending.length;
  });
}
```

This holds a transaction across network I/O — acceptable *only* because the batch is bounded and every publish has a deadline; size the batch so the transaction stays short. And the failure unit is the batch, not the row: the `publishedAt` marks commit with the transaction, so a crash (or one failed publish) at row 50 rolls back the marks for rows 1–49 too, and the next pass republishes the whole batch head. The claim-then-publish alternative — a status column, each row marked in its own transaction — shrinks redelivery to single rows at the cost of more bookkeeping. Either way the trade is deliberate: **at-least-once delivery, never zero-times**.

Polling on a short interval is the simple relay. Postgres `NOTIFY` on insert can wake it early, and CDC log-tailing (Debezium-style) replaces polling entirely at higher volume — the table contract stays identical either way.

### At-least-once, therefore consumers dedupe

The relay redelivers on crash, so every event carries the outbox row's `id` as `eventId`, and consumers claim it before applying side effects — the consumer side of `idempotency-keys-on-writes`. An outbox without consumer dedupe swaps "lost events" for "double-applied events"; you need both halves.

### Ordering

`SKIP LOCKED` plus concurrent relays means global order is not guaranteed across batches. If consumers need per-entity order (all events for one invoice in sequence), either run a single relay instance, or partition the claim by an aggregate key so one worker owns each entity's stream. Decide this before the first consumer assumes ordering it isn't getting.

### Keep the outbox bounded

Published rows are done — sweep them on a retention schedule (`steady-state-purge-unbounded-growth`), and alert on the age of the oldest *unpublished* row: outbox lag is the one metric that says "commits are happening but the world isn't hearing about them".

## Pressure Resistance

### "The publish almost never fails"

The publish isn't the main risk — the *gap* is. Deploys restart your process many times a day, and each restart is a chance to die between commit and publish. "Almost never" times every deploy times every handler is a steady trickle of silently missing events.

### "We try/catch the publish and log it"

Then the failure mode is: write committed, event lost, one log line nobody is paged on. A log is not a durable, replayable record of what you owe consumers. The outbox row is exactly that record — and it costs the same one insert.

### "A nightly re-sync will catch drift"

Re-sync reconstructs *current state*, not the transitions. `invoice.refunded` at 2pm followed by `invoice.paid` at 3pm looks identical to "paid all along" by midnight. Deltas can't be recovered from a snapshot; only the event you persisted at commit time carries them.

### "We'll use two-phase commit with the broker"

Most brokers, webhook targets, and HTTP APIs don't participate in your database's transaction, and distributed transactions across the ones that could are operationally heavier than one extra table. The outbox gets atomicity from the single system that already has it — your database.

### "It adds a table and a worker for a simple feature"

One table, one insert per event, one bounded loop. Compare against the incident where finance asks why the ledger and the orders table disagree, and the answer is "a deploy at 14:02 ate the event". The machinery is the cheap side.

## Red Flags

- `await db.update(...)` followed by `await publishEvent(...)` with no outbox between them.
- `publishEvent` / `fetch` to a webhook inside a `db.transaction` callback.
- A `catch` around a publish that logs and continues.
- Events built from *current* entity state at publish time instead of from the payload captured at commit time.
- No `eventId` on published events — consumers can't dedupe the relay's redeliveries.
- An outbox table with no purge and no lag alert.
- A reconciliation cron described as the safety net for missing events.

**All of these mean: the write and the announcement can diverge — put the event in the transaction and let a relay deliver it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The broker is HA, publishes don't fail" | Your process is not HA between two awaits. The gap, not the broker, loses the event. |
| "We log publish failures" | A log line is not delivery and not replayable. The outbox row is both. |
| "Nightly sync covers it" | Snapshots can't reconstruct transitions. Refund-then-pay looks like paid-all-along. |
| "Publish inside the transaction, then it's atomic" | The broker can't roll back. A failed commit after a successful publish is a phantom event. |
| "At-least-once will double-process" | That's what consumer dedupe on `eventId` is for — and it's required with or without an outbox. |
| "It's over-engineering for our volume" | The failure is per-deploy, not per-request. Low volume just means each lost event matters more. |

## Related

- `idempotency-keys-on-writes` — the other half: the relay delivers at-least-once, so consumers dedupe on the outbox `eventId`. That skill is consumer-side receipt dedup; this one guarantees the send exists at all.
- `dead-letter-and-replay` — the failure *after* delivery: a consumer whose handler throws. The outbox covers the gap before the broker ever sees the event; dead-lettering covers what happens once it does.
- `retry-with-jitter-and-budget` — the relay's publish attempts follow that discipline.
- `steady-state-purge-unbounded-growth` — published rows need a retention sweep.

## Reference

- Chris Richardson, [*Pattern: Transactional outbox*](https://microservices.io/patterns/data/transactional-outbox.html) — the canonical statement of the pattern and the dual-write problem it solves.
- AWS Prescriptive Guidance, [*Transactional outbox pattern*](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html) — the pattern in AWS's cloud design patterns catalog.
- Gunnar Morling, [*Reliable Microservices Data Exchange With the Outbox Pattern*](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/) (Debezium blog, 2019) — the CDC-based relay variant.
