---
name: retry-with-jitter-and-budget
description: Use when designing client code that calls an external dependency (HTTP, DB, third-party SDK, LLM). Use when adding `retry: 3` to a library. Use when seeing thundering-herd patterns in dashboards (sync retries causing correlated traffic spikes). Use when reviewing code that retries non-idempotent writes without an idempotency key.
---

# Retry With Jitter and Budget

## Overview

**Retries use exponential backoff with full jitter and a documented attempt count.** Never retry a non-idempotent write without an idempotency key. Never retry indefinitely. Never retry without budgeting the total deadline.

A retry without these three (jitter, budget, idempotency) makes things worse, not better.

## The Iron Rule

```
NEVER retry without jitter, a deadline, and idempotency safety. Naked retries cause outages.
```

**No exceptions:**
- Not for "I'll just retry 3 times with a 1-second delay"
- Not for "the dependency rarely fails"
- Not for "the library has retries built-in"
- Not for "infinite retries are fine for jobs that must succeed"

## Why

Retries are deceptive — they feel like resilience but, done wrong, they *cause* outages:

- **Thundering herd.** Synchronized retries (every client retrying at exactly 100ms, 300ms, 700ms) hit the recovering dependency in waves, prolonging the outage. Jitter spreads them.
- **Cost explosion.** Retries on a slow dependency multiply your load. Three retries on a 5s dependency = 20s of work per request, at 4× the rate. Budget caps the total.
- **Duplicate side effects.** Retrying `POST /charge` without idempotency = double-charging customers. Idempotency keys make retries safe.
- **Stack starvation.** Unbounded retries hold workers and connections; eventually the system can't accept new work.

The discipline isn't "retry less" — it's *retry with the right shape*. Three properly-jittered retries with a budget recover gracefully; three eager retries take a system down.

## Detection

You are violating the rule if any of these are true:

- A retry loop with a fixed delay (`for (let i = 0; i < 3; i++) { await sleep(1000); }`).
- A retry on `POST`/`PUT`/`PATCH` without an idempotency key on the request.
- A library setting like `maxRetries: 3` with no jitter configuration.
- A retry policy with neither an overall deadline *nor* a bounded attempt count with capped backoff ("retry until success").
- Retries layered: SDK retries 3×, your wrapper retries 3×, the framework retries 3× → 27× actual attempts.
- A "transient error" handler that doesn't classify the error before retrying.

## The Pattern

### The canonical retry loop

```ts
type RetryOptions = {
  maxAttempts: number;        // including the first attempt
  baseDelayMs: number;        // first backoff
  maxDelayMs: number;         // cap
  deadlineMs: number;         // total budget across all attempts
  isRetryable: (err: unknown) => boolean;
  signal?: AbortSignal;
};

export async function withRetry<T>(
  op: (attempt: number) => Promise<T>,
  opts: RetryOptions,
): Promise<T> {
  const deadline = Date.now() + opts.deadlineMs;
  let lastErr: unknown;

  for (let attempt = 1; attempt <= opts.maxAttempts; attempt++) {
    if (opts.signal?.aborted) throw new Error('aborted');
    if (Date.now() >= deadline) break;

    try {
      return await op(attempt);
    } catch (err) {
      lastErr = err;
      if (!opts.isRetryable(err)) throw err;
      if (attempt === opts.maxAttempts) break;

      // Exponential backoff with full jitter.
      const expBackoff = Math.min(opts.maxDelayMs, opts.baseDelayMs * 2 ** (attempt - 1));
      const delay = Math.random() * expBackoff; // uniform over [0, expBackoff]
      const remaining = deadline - Date.now();
      if (delay >= remaining) break;
      await new Promise(r => setTimeout(r, delay));
    }
  }

  throw lastErr;
}
```

The three properties on display:

1. **Jitter:** `Math.random() * expBackoff` — *full jitter*. Don't use "decorrelated jitter" or "equal jitter" — full is simplest and works for almost every workload.
2. **Budget:** `deadlineMs` caps total time across all attempts. The function bails when the budget is exhausted, even if attempts remain. (A budget can also be *bounded attempts × a capped per-attempt delay* — `maxAttempts` + a `maxTimeoutInMs` ceiling on backoff — which is the common shape in queue/task-runner config where there's no single request to deadline. Both bound total work; the anti-pattern is having *neither*.)
3. **Classification:** `isRetryable(err)` — only certain errors retry. Programming bugs (4xx, validation errors) should never retry.

### Using it — read with timeout

```ts
const data = await withRetry(
  () => fetch('https://api.example.com/data', {
    signal: AbortSignal.timeout(5_000),
  }).then(r => {
    if (r.status >= 500) throw new RetryableError(`upstream ${r.status}`);
    if (!r.ok) throw new Error(`bad request ${r.status}`); // not retryable
    return r.json();
  }),
  {
    maxAttempts: 3,
    baseDelayMs: 200,
    maxDelayMs: 2_000,
    deadlineMs: 10_000,
    isRetryable: (err) => err instanceof RetryableError,
  }
);
```

The outer `withRetry` has a 10s budget. Each attempt has a 5s timeout. The two compose: any one attempt can take up to 5s; the total never exceeds 10s.

### Write with idempotency key — same key across retries

```ts
const idempotencyKey = crypto.randomUUID();

await withRetry(
  () => fetch('/api/orders', {
    method: 'POST',
    headers: { 'idempotency-key': idempotencyKey, 'content-type': 'application/json' },
    body: JSON.stringify(input),
    signal: AbortSignal.timeout(5_000),
  }),
  {
    maxAttempts: 3,
    baseDelayMs: 200,
    maxDelayMs: 2_000,
    deadlineMs: 15_000,
    isRetryable: isNetworkOr5xx,
  },
);
```

The same `idempotencyKey` is reused across retries. The server dedups. The retry is now safe.

### What's retryable, what isn't

| Class | Retry? | Why |
|---|---|---|
| Network errors (`ECONNRESET`, `ETIMEDOUT`, `AbortError`) | Yes | Transient. Request didn't reach the server, or response didn't return. |
| HTTP 5xx | Yes | Server-side transient. May resolve. |
| HTTP 429 (rate limit) | Yes, honor `Retry-After` | Explicit signal to back off. |
| HTTP 408 (request timeout) | Yes | Transient. |
| HTTP 4xx (other) | No | Client bug. Retrying produces the same error. |
| HTTP 401 / 403 | No | Auth issue. Retrying without changing creds is futile. |
| Schema validation errors | No | Programming/client bug. |
| OOM, disk full | No | Throwing them around is worse. |
| `AbortError` from your own timeout | Yes (subject to budget) | The timeout fired; underlying op might be transient. |

### Honoring `Retry-After`

The table says to honor `Retry-After` on 429 — but the canonical loop above can't, because `isRetryable` returns only a boolean and the delay is purely client-computed. Give the classifier a way to surface the server's value:

```ts
// isRetryable can return a suggested delay, not just true/false.
const retryAfterMs = (err: unknown): number | undefined => {
  const h = (err as { response?: { headers?: { get?: (k: string) => string | null } } })
    .response?.headers?.get?.('retry-after');
  return h ? Number(h) * 1000 : undefined;   // seconds → ms
};

// In the loop: server value wins, still capped by `remaining` (the budget left, from the canonical loop above).
const delay = Math.min(retryAfterMs(err) ?? Math.random() * expBackoff, remaining);
```

On a 429/503 carrying `Retry-After` (or `RateLimit-Reset`), the server is telling you exactly when it'll be ready — that value beats client backoff. Without a hook for it, "honor `Retry-After`" is advice the loop can't follow.

### Disable library-level retries

Most SDKs retry internally. To prevent layering, disable them and let your wrapper control it:

```ts
const payments = new PaymentClient(env.PAYMENT_KEY, {
  timeoutMs: 10_000,
  maxNetworkRetries: 0, // ← you control retries
});

const fetcher = createFetch({ retry: 0 });
```

This keeps the retry semantics in one place. Three layers of retries is the same as 27 unwanted attempts.

The rule is *one* retry layer, not *zero* library retries — and that layer isn't always your wrapper. When a durable task runner or queue (Trigger.dev, Inngest, SQS + worker) owns the retry lifecycle — re-invoking the whole handler with its own backoff, jitter, and budget — *that* is the single retry layer, and the leaf call inside the handler must set retries to 0. Pick the layer that owns retries; everywhere else, disable them. The failure mode is two owners silently stacking.

### Which jitter strategy

The data (Marc Brooker, AWS Builders' Library) compares four:
- **No jitter** — thundering herd, worst.
- **Equal jitter** — `(exp/2) + random(0, exp/2)` — good.
- **Full jitter** — `random(0, exp)` — best in most workloads; lowest server load.
- **Decorrelated jitter** — `random(base, prev*3)` — best when retries are highly correlated.

Default to **full jitter** unless you have measurement saying otherwise.

## Pressure Resistance

### "I'll just retry 3 times with a 1-second delay"

Fixed delays cause thundering herd. Every client retrying at exactly the same offset hits the recovering dependency in synchronized waves. Jitter is non-optional.

### "The dependency rarely fails, retries aren't worth the complexity"

When the dependency *does* fail, fixed retries make the outage worse. Jittered retries make it recoverable. The complexity is a one-time library cost; the payoff is every future incident.

### "The library has retries built-in"

Disable them. Library retries layer with yours and produce surprising attempt counts. Centralize retry semantics in your code.

### "An infinite retry loop is fine for jobs that must succeed"

No retry is infinite. Even "must eventually succeed" jobs need a deadline; what changes is *what happens at the deadline* (move to dead-letter queue, escalate to oncall, mark as failed). Unbounded retries hold workers and pool entries indefinitely.

### "Retrying a non-idempotent endpoint without a key is fine for low traffic"

Until it isn't. The double-charge or duplicate-row bug is hard to trace and impossible to undo. Idempotency keys cost one line on the request; their absence costs customer trust.

## Red Flags

- A retry loop with `await sleep(constant)` between attempts.
- A retry on `POST`/`PUT` with no `Idempotency-Key` header.
- A while-loop retry with no maximum attempt count.
- An infinite-retry job processor.
- `Promise.all` of retried calls — turns "one slow dependency" into "all of them retried in parallel."
- A timeout dashboard showing flat-topped error spikes (synchronized retries).
- A code review where "how many actual attempts" can't be answered.

**All of these mean: the retry will make the outage worse — add jitter, budget, and idempotency before shipping.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Fixed delay is simpler" | Until your retries take the dependency down. |
| "The default jitter in lib X is fine" | Check it. Many libraries default to no jitter or "equal jitter" (slightly worse than full). |
| "It's an internal service, dedup is overkill" | Internal services have outages too. Their clients retry. |
| "Budget is for paranoids" | Budget is what saves you from a stuck retry holding a connection forever. |
| "Library retries are good enough" | Fine *if that layer is the single source of retry truth* and configures jitter + budget + bounded attempts (e.g. a durable task runner). Bad when it silently layers *under* your own loop — then they compound invisibly. |
| "We'll add jitter when we see herd problems" | The herd problem looks like a flat-line at saturation. Add jitter now. |

## Related

- `timeouts-everywhere` — each attempt is timed; the retry has a total budget
- `circuit-breaker-on-flaky-deps` — handoff: transient retries vs. sustained-outage breaking
- `idempotency-keys-on-writes` — retrying writes needs idempotency keys

## Reference

- Marc Brooker (AWS), [*Timeouts, retries, and backoff with jitter*](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — the canonical write-up, including the comparison of jitter strategies.
- Marc Brooker, [*Exponential Backoff And Jitter*](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) (AWS Architecture Blog, 2015) — the original blog post with the measured data.
- Michael Nygard, *Release It!* 2e (2018), ch. 5 — pairs retries with circuit breakers and bulkheads.
