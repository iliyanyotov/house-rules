---
name: idempotency-keys-on-writes
description: Use when a consequential write may be retried by a client, load balancer, queue, webhook sender, or double-click. Use when a payment, order, or signup endpoint lacks an idempotency key, or duplicate rows/effects appear in a bug report.
---

# Idempotency Keys on Writes

## Overview

**Every external-facing mutating endpoint accepts an `Idempotency-Key` header** (or equivalent client-supplied token), and the result is the same whether the request is sent once or one hundred times within the retention window.

The retention window is at least 24 hours. The key is client-supplied. The implementation is server-side dedup keyed by `(endpoint, key)`.

## The Iron Rule

```
NEVER expose a *consequential* mutating endpoint without idempotency. Networks lose responses; clients retry. (Low-stakes notifications can use a lighter UI-guard + short-window dedup — see "Low-stakes notifications".)
```

**No exceptions:**
- Not for "it's overkill for our scale"
- Not for "clients won't retry"
- Not for "our DB has unique constraints"
- Not for "we'll add it later"

## Why

Networks lose responses. Clients retry on timeout. Queues redeliver. Load balancers re-issue. Users double-click. A mutating endpoint that doesn't dedupe will, eventually, charge a card twice, create two orders, send two welcome emails.

The fix is not "be more careful." It's structural: **let the client name each unique intent and let the server enforce one execution per name.** The client retries safely; the server dedupes deterministically.

## Detection

You are violating the rule if any of these are true:

- A mutating endpoint accepts no client-supplied dedup token.
- A handler enqueues a job, inserts a row, or sends an email without checking "did we already do this?"
- A webhook handler processes the same provider event twice and produces two writes.
- A client retries on `5xx` and a duplicate side effect is the result.
- A user can double-click submit and end up with two records.
- A queue with at-least-once delivery has consumers that aren't idempotent.

## Two stances: producer side and consumer side

Idempotency shows up in two situations, and they differ in *who owns the key*:

- **Producer side — you expose a mutating endpoint.** You *require* the caller to supply a key (`Idempotency-Key` header) and dedupe on `(endpoint, key)`. You set the contract. Covered by *Server-side* and *Client-side* below. (A server-*derived* key — e.g. hashing a booking's time window — can stop a narrow class of duplicates, but it is *not* a substitute for a client key: the caller still can't retry and get the *original* response back, and you can't dedupe two genuinely-distinct intents that happen to hash the same. If you can't expose a header, say so in the API docs and document the retry semantics.)
- **Consumer side — you receive an at-least-once stream you don't control** (a provider webhook, a queue, a pub/sub topic). You *don't* get to demand a key — you **derive** one from something naturally stable the producer already sends (the event ID, the message ID), and dedupe on it before applying the side effect. Covered by *Webhooks*, *Queue consumers*, and *Scheduled jobs* below.

The consumer side has a second failure mode the producer side doesn't: a delivery can not just *duplicate* but *fail to process at all*. Dedupe (this skill) stops a duplicate from double-applying; it does **not** save an event whose handler throws. For that — persisting the failed event and replaying it — see `dead-letter-and-replay`. The two compose: dedupe on arrival, dead-letter on failure.

## The Pattern

### Server-side: reserve the key before the side effect

```ts
async function handleCreateOrder(req: Request) {
  const endpoint = 'POST /orders';
  const key = req.headers.get('idempotency-key');
  if (!key) return json({ error: 'missing_idempotency_key' }, { status: 400 });

  // Parse BEFORE reserving — a malformed body must never strand a key in `processing`.
  const body = CreateOrder.parse(await req.json());
  // Bind the key to the exact request, so a reused key with a different body is caught, not replayed.
  const requestHash = sha256(canonicalJson(body));

  // Atomic insert: the unique `(endpoint, key)` decides who executes. The short lease
  // makes the reservation reclaimable if the owner dies mid-flight.
  const reservation = await idempotencyKeys.reserve({
    endpoint,
    key,
    requestHash,
    leaseUntil: new Date(Date.now() + 30_000),
    expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000),
  });

  if (reservation.status === 'completed') {
    if (reservation.requestHash !== requestHash) {
      return json({ error: 'idempotency_key_reuse' }, { status: 422 }); // same key, different intent
    }
    return replay(reservation);
  }

  if (reservation.status === 'processing') {
    // Reject a different intent immediately; do not wait for unrelated work to finish.
    if (reservation.requestHash !== requestHash) {
      return json({ error: 'idempotency_key_reuse' }, { status: 422 });
    }
    // Someone else holds a live lease. Wait with a DEADLINE — never block forever.
    const completed = await idempotencyKeys.waitForCompletion({ endpoint, key }, { timeoutMs: 5_000 });
    if (!completed) return json({ error: 'processing', retryAfter: 1 }, { status: 409 });
    if (completed.requestHash !== requestHash) return json({ error: 'idempotency_key_reuse' }, { status: 422 });
    return replay(completed);
  }

  // status === 'owned' — a fresh insert, or we reclaimed an expired lease. `generation`
  // is a monotonically increasing fencing token: a stale owner can never complete generation N
  // after a reclaimer has advanced the row to N+1.
  const generation = reservation.generation;
  const result = await db.transaction(async (tx) => {
    // Lock the reservation row and prove we still own this generation in the SAME transaction
    // as the business write. A concurrent reclaimer either ran first (this returns false) or
    // waits for this transaction and then observes `completed`.
    const stillOwned = await tx.idempotencyKeys.lockOwned({ endpoint, key, generation });
    if (!stillOwned) throw new Error('idempotency_ownership_lost');

    const order = await tx.orders.create(body);
    const responseBody = JSON.stringify({ order });

    await tx.idempotencyKeys.complete({
      endpoint,
      key,
      generation,
      responseBody,
      responseStatus: 201,
    });

    return { responseBody, responseStatus: 201 };
  });

  return new Response(result.responseBody, { status: result.responseStatus });
}
```

Six properties make this safe, and dropping any one reintroduces a duplicate or a hang: **parse before reserve** (a bad body can't strand a key), **bind the key to a `requestHash`** (a reused key with a different body is rejected, not silently replayed), **a processing lease** (a dead owner's key is reclaimable, not stuck forever), **a fencing generation** (a reclaimed stale owner cannot commit), **a bounded wait** (a slow owner yields a `409` retry, not an infinite block), and **ownership verification + side effect + `complete` in one transaction** (they commit or roll back together under the reservation-row lock). A long database transaction holds that row lock, so reclamation waits and then sees completion. External effects that *can't* join that transaction — an email send, a third-party charge — go through an outbox row written *inside* the transaction and dispatched after commit (forwarding an idempotency key downstream; see *Outbound calls*). A lease without fencing is unsafe: expiry alone does not stop the old worker.

### Concurrency — handle the race deterministically

```ts
try {
  await idempotencyKeys.insertProcessing({ endpoint, key }); // succeeds → we execute
} catch (err) {
  if (isUniqueViolation(err)) {
    // Someone else is doing or did the work. Wait with a deadline, then 409 if still pending.
    const completed = await idempotencyKeys.waitForCompletion({ endpoint, key }, { timeoutMs: 5_000 });
    if (!completed) return json({ error: 'processing', retryAfter: 1 }, { status: 409 });
    return replay(completed);
  }
  throw err;
}
```

Use the database's atomic insert/upsert (`INSERT ... ON CONFLICT DO NOTHING RETURNING *`) for the reservation. Never do `find` → side effect → `insert`; that is the duplicate-write race.

### Client-side: generate the key once per intent

```ts
async function createOrder(input: CreateOrderInput): Promise<Order> {
  const idempotencyKey = crypto.randomUUID();
  return withRetry(() =>
    fetch('/api/orders', {
      method: 'POST',
      headers: {
        'content-type': 'application/json',
        'idempotency-key': idempotencyKey,
      },
      body: JSON.stringify(input),
      signal: AbortSignal.timeout(10_000),
    }),
  );
}
```

The key is generated *once per intent*. Retries reuse the same key, so the server dedupes.

### Webhooks — dedup on the provider's event ID

Most providers send a unique `id` on every event and retry on non-200 responses. Use that ID as the idempotency key:

```ts
async function handlePaymentWebhook(req: Request) {
  const event = PaymentEvent.parse(await req.json());

  const processed = await db.transaction(async (tx) => {
    const claimed = await tx.processedWebhooks.insertOnce({ eventId: event.id });
    if (!claimed) return false;

    await handlePaymentEvent(tx, event);
    await tx.processedWebhooks.markCompleted({ eventId: event.id, processedAt: new Date() });
    return true;
  });

  return json({ ok: true, replay: !processed });
}
```

The claim and the side effects share a transaction. A duplicate delivery loses the unique-key race before it can perform duplicate work.

### Queue consumers — message ID as key

For at-least-once queues, the message ID is the natural key. Each consumer claims the message ID with a unique insert before doing work, and marks it completed in the same transaction as the side effect.

### Outbound calls — forward an idempotency key downstream

Dedup protects *your* writes; it does nothing for the third-party side effects your handler triggers. When you call a mutating external API (Stripe, an email provider) inside a handler that may itself be retried, pass *that SDK's* idempotency-key option — derived from the same intent key — so a retry of your handler doesn't charge the card or send the email twice:

```ts
// ✅ Stripe re-executes only if the key is new; your retry reuses the original charge.
await stripe.paymentIntents.create(params, { idempotencyKey: `charge:${intentKey}` });
```

### Scheduled jobs — the natural-time key

Cron-driven writes have no client to supply a header. They have something better: a *naturally stable* key — the date the cron fires for, the hour bucket, the batch identifier:

```ts
async function ingestDailyFeed() {
  const todayIso = new Date().toISOString().slice(0, 10);

  // "Already doing or done today?" — the date IS the idempotency key.
  const claimed = await dailyImports.claimDate(todayIso);
  if (!claimed) {
    return json({ ok: true, skipped: 'already_ingested', date: todayIso });
  }

  await ingest(todayIso);
  await dailyImports.markCompleted(todayIso);
  return json({ ok: true, date: todayIso });
}
```

The claim is a unique insert. A retry within the same time window hits the existing row and skips or waits, instead of running a second import.

The rule generalizes: **the idempotency key is whatever is naturally stable for the unit of work.** For HTTP it's a client-supplied UUID. For cron it's the time bucket. For webhooks it's the provider's event ID. For queues it's the message ID.

A unique *constraint* on that key only **rejects** duplicates with an error — that's a collision guard, not idempotency. True idempotency means catching the unique-violation and **returning the prior result** (skip-and-replay), not surfacing a 409 to a caller that is simply retrying. The constraint stops the double-write; the catch-and-return is what makes the retry *succeed*.

### Low-stakes notifications — UI guard + short-window dedup

Idempotency is mandatory when duplicates are *consequential* — money moves, data is created, an order ships twice. For low-stakes notifications (contact-form auto-reply, "thanks for signing up" email), the cost of an occasional duplicate is small. A UI-level double-submit guard plus a short-window payload hash is often enough.

| Operation | Discipline |
|---|---|
| Payment, order, account creation | Mandatory header-keyed idempotency table |
| Webhook from a provider | Mandatory event-ID dedup |
| Cron-triggered write | Mandatory naturally-stable-key precheck |
| Contact-form notification | UI guard + short-window payload dedup |
| `PUT` (pure overwrite) | Naturally idempotent |
| `PATCH` | Only when it's a pure overwrite — RFC 9110 doesn't guarantee PATCH idempotency (a JSON-Patch array append duplicates on retry) |
| `DELETE` | Naturally idempotent |
| `GET` | Naturally idempotent — no key needed |

## Pressure Resistance

### "It's overkill for our scale"

Scale doesn't matter. Networks lose responses at every scale. A double-charge at 10 RPS is just as bad as at 10K RPS — worse, even, because the small-scale ops team will see it as "weird, this happens randomly" and miss the structural cause.

### "Clients won't retry, so dedup is unnecessary"

Clients retry. They just retry in places you don't see. Browsers retry on network errors. Service workers retry. Load balancers retry. Mobile apps retry. Webhook senders retry. The "trust the client not to retry" model has lost in every system ever built.

### "Our DB has unique constraints, that's enough"

Unique constraints catch *some* duplicates. They don't catch: two `POST`s that should both be valid (two different orders for the same customer); side effects outside the DB (email sends, queue pushes, API calls); the case where the original request succeeded but the response was lost — the constraint rejects the retry but the client doesn't know the first succeeded.

### "We'll have ballooning storage"

Set an expiry (24h is a common default). Auto-clean expired rows nightly. The volume is your write rate × 24h × a few hundred bytes per record. Manageable at any normal scale.

### "It complicates the API"

One header. The client passes a UUID. The server stores it. The complication is on the server, where the team owns and tests it. The API gains one optional header (which becomes mandatory in v2).

## Red Flags

- A mutating endpoint with no idempotency-key handling.
- A client that retries on 5xx without sending a stable key.
- A webhook handler that doesn't check the provider's event ID.
- A queue consumer that doesn't dedupe on message ID.
- A retried handler that re-calls a payment/email SDK without passing that SDK's idempotency-key option — your dedup is intact, the downstream charge/send fires twice.
- A unique-constraint violation surfaced to a retrying caller as a 409 instead of caught-and-replayed — a collision guard mistaken for idempotency.
- A bug ticket containing "we saw two records when we expected one."
- A "fix" that adds a debounce or throttle to a UI button — that's hiding the bug.
- The phrase "this should only happen once" with no enforcement mechanism.

**All of these mean: dedup is missing — name the key and store it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The framework prevents retries" | No framework prevents network-level retries. The HTTP layer alone can replay. |
| "We use HTTPS, packets don't get lost" | HTTPS doesn't prevent lost *responses*; it prevents reading them. The retry happens anyway. |
| "It's only the happy path that matters" | The unhappy path is exactly where idempotency earns its keep. |
| "Unique constraints + transactions = idempotent enough" | For DB-only side effects, almost. For anything beyond the DB, not enough. |
| "Adding the table is painful" | Two columns and a composite primary key. A 30-second migration. |
| "Stripe-style is too heavyweight" | The implementation above *is* Stripe-style. No frameworks, no services. |

## Related

- `retry-with-jitter-and-budget` — makes retried writes safe
- `dead-letter-and-replay` — consumer-side dedup so replay is safe
- `race-conditions` — prevents double-submit races
- `outbox-for-atomic-write-and-publish` — the producer side: the event exists because it committed with the write; this is the consumer side that dedupes its redelivery

## Reference

- Stripe Engineering, [*Designing robust and predictable APIs with idempotency*](https://stripe.com/blog/idempotency) — the canonical industry write-up.
- Malcolm Featonby (AWS), [*Making retries safe with idempotent APIs*](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/) — the Amazon Builders' Library treatment of idempotency tokens as the mechanism that makes retries safe.
- Consumer-side companion: `dead-letter-and-replay` handles the *other* failure mode of an at-least-once stream — an event whose handler throws — which dedupe alone does not cover.
