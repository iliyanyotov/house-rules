---
name: cap-async-fan-out
description: Use when `Promise.all(items.map(asyncFn))` runs over an array whose length you don't control. Use when a batch job fires one request per row of a growing table. Use when a fan-out exhausts the DB connection pool, trips 429s, or OOMs under load. Use when unrelated requests slow down whenever a bulk operation runs. Use when choosing between `Promise.all` and `Promise.allSettled` for a multi-item operation.
---

# Cap Async Fan-Out

## Overview

**Async work fanned out over input you don't control gets an explicit concurrency cap.** `items.map(asyncFn)` is not a work queue — it *starts* every operation on the same tick. `Promise.all` doesn't schedule anything; it only waits for what `.map` already unleashed.

Ten items today is ten thousand after growth. The fan-out that was invisible in dev becomes the thing that drains the connection pool for every other request in the process.

## The Iron Rule

```
NEVER fan out async work with unbounded concurrency over input whose
size you don't control. Bound the parallelism with an explicit cap.
```

**No exceptions:**
- Not for "it's fast, the items are small"
- Not for "the provider will just rate-limit us"
- Not for "it's a one-off backfill script"

**Fixed, small fan-outs are already excluded by the rule's own scope — "input whose size you don't control" — not excepted from it:** `Promise.all([getUser(id), getOrders(id)])` fans out over a set whose size is written in the code — two. The rule triggers on *input-sized* fan-out (`rows.map`, `users.map`), where the cardinality belongs to production data, not to you.

## Why

Every `asyncFn` starts eagerly, so the instantaneous concurrency equals the array length. Three resources pay for that:

1. **Your connection pool.** 10k queries racing for a 10-connection pool means the fan-out monopolizes the pool for its whole duration — every *unrelated* request in the process queues behind it. A documented production shape: the "fast" endpoints suddenly take seconds whenever the bulk path runs, because they're waiting for a connection the fan-out holds hostage.
2. **The downstream.** 10k simultaneous calls is a self-inflicted load spike on the API you depend on: 429s, then your retries (see `retry-with-jitter-and-budget`) amplify it.
3. **Your memory.** Every in-flight operation holds its request, response buffer, and closure live at once.

```ts
// ❌ Concurrency = rows.length. 40 rows in dev; 40,000 in prod.
const results = await Promise.all(
  rows.map((row) => enrichFromApi(row)),
);

// ✅ Same work, at most 5 in flight at any moment.
import pLimit from 'p-limit';

const limit = pLimit(5);
const results = await Promise.all(
  rows.map((row) => limit(() => enrichFromApi(row))),
);
```

The cap number is a tuning decision (start small — single digits — and raise with evidence), but *having* a cap is not.

## Detection

You are violating the rule if any of these are true:

- `Promise.all` / `allSettled` over `.map` on data from a query, a request body, a file, or a queue — anything whose length grows with usage.
- A loop that pushes promises into an array without awaiting anything until the end.
- Pool-exhaustion symptoms correlated with a bulk path: unrelated endpoints slow down when the import/backfill/notification job runs.
- A burst of 429s from a provider each time a batch feature fires.
- A "process all users" script with no concurrency control beyond hope.

## The Pattern

### A limiter, not chunks

`p-limit` (or `p-map`, which composes map+limit) keeps a fixed number in flight and starts the next item the moment one finishes. Chunking (`for` over slices of 10 with `Promise.all` per slice) is the common hand-rolled substitute, but it stalls on stragglers: the batch waits for its slowest member before starting the next slice. A limiter maintains full utilization at the cap.

```ts
// Plain-TS limiter shape, if you'd rather not add the dependency:
async function mapWithLimit<T, R>(
  items: readonly T[],
  limit: number,
  fn: (item: T) => Promise<R>,
): Promise<R[]> {
  if (limit < 1) throw new RangeError('limit must be >= 1'); // 0 would silently do nothing
  const results = new Array<R>(items.length);
  let next = 0;
  async function worker(): Promise<void> {
    while (next < items.length) {
      const i = next++;                 // single-threaded: no race on the counter
      results[i] = await fn(items[i]!); // one in flight per worker
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}
```

### Choose `all` vs `allSettled` deliberately

- **`Promise.all`** rejects on the first failure — but the *remaining* work keeps running, unwatched by you (their rejections are observed by `all` internally; the operations themselves are not cancelled). If abandoning the batch on first failure is the intent, pair the fan-out with an `AbortSignal` so "abandoned" also means "stopped."
- **`Promise.allSettled`** runs everything to completion and reports per-item outcomes — right for independent items where one failure shouldn't sink the rest (see `graceful-degradation-defaults` for the criticality split). It also *waits* for everything, so per-item timeouts (`timeouts-everywhere`) still apply.

Pick the one whose failure semantics you actually want; don't default to `all` because it's shorter.

### The cap is per instance

A module-level limiter caps *this process*. Under autoscaling, aggregate downstream pressure is cap × instance count — the same honesty `bulkhead-isolated-failure-domains` demands of its in-flight caps. For a shared downstream with a hard global budget, the real limit must live in a shared place (a queue with worker concurrency, a rate limiter service), not in per-instance arithmetic.

### First ask: should this be a fan-out at all?

If each item's work is a *query*, the fix is often one query — `WHERE id IN (...)`, a join, a batched upsert — not N capped queries. That's `n-plus-one-prevention`'s territory: bound the query *count* first; cap the concurrency of whatever legitimately remains fan-out (external APIs, emails, per-item side effects that can't batch).

## Pressure Resistance

### "It's only ever a handful of items"

Then the array's size is a domain fact — encode it (a fixed tuple, a domain-bounded set) and the rule doesn't trigger. If the honest answer is "however many rows the query returns", the size belongs to production, and production grows.

### "The pool/provider will throttle us anyway"

Being throttled *is* the incident: your unrelated requests queue behind the fan-out, and provider 429s turn into retry storms. A cap keeps the pressure below the cliff instead of discovering the cliff.

### "A cap makes the batch slower"

Slightly slower for the batch; dramatically better for everything sharing the process and the downstream. If batch latency matters, raise the cap with measurements — the difference between 5 and unbounded is rarely the batch's bottleneck, and never worth the pool.

### "It's a one-off script"

One-off scripts run against production data with production credentials, and are exactly where "process all users" lives. The limiter is three lines.

## Red Flags

- `Promise.all(rows.map(...))` where `rows` came from a query or request.
- `await` inside `.map(...)` — the map produces promises either way; the await does nothing to bound them.
- A backfill script whose concurrency section is absent.
- Pool timeout errors that correlate with a scheduled job.
- A hand-rolled chunk loop presented as the concurrency fix.

**All of these mean: bound the parallelism — a limiter with a small cap, or restructure into one batched query.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Node is single-threaded, it can't overload anything" | The event loop is single-threaded; the 10k sockets and pool checkouts it opens are not. |
| "We tested with realistic data" | Realistic today. The fan-out scales with the table; the pool doesn't. |
| "allSettled means failures are handled" | It handles *rejections*, not load. Concurrency is orthogonal to failure semantics. |
| "We need it as fast as possible" | Unbounded isn't fastest — it's pool-thrashing, retry-amplifying, and OOM-prone. Tune the cap. |
| "The ORM manages connections for us" | It manages a *pool* — a fixed budget your fan-out spends all at once. |

## Related

- `n-plus-one-prevention` — first collapse per-item queries into one batched query; cap the concurrency of what legitimately remains a fan-out.
- `shed-load-under-overload` — that skill sheds *inbound* work to protect you; this caps *outbound* work to protect your pool and your downstreams.
- `bulkhead-isolated-failure-domains` — bulkheads partition caps across *different* dependencies; this bounds parallelism *within one* operation. Both caps are per-instance.
- `timeouts-everywhere` — every fanned-out item still needs its own deadline; a cap without timeouts just queues the hang.
- `no-floating-promises` — the sibling rule: this bounds how many promises you start; that one guarantees every started promise is observed.

## Reference

- [MDN — `Promise.all()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all) — semantics: it awaits already-started promises; it does not schedule or throttle them.
- [sindresorhus/p-limit](https://github.com/sindresorhus/p-limit) — the standard limiter; [p-map](https://github.com/sindresorhus/p-map) composes map + limit with a `concurrency` option.
- Matt Burke, ["Promise.all is too much of a good thing"](https://www.mattburke.dev/promise-all-is-too-much-of-a-good-thing/) (2024) — the connection-pool-exhaustion incident write-up behind the Why section: 50+ queries through `Promise.all` drained a 10-connection pool and queued every unrelated request.
