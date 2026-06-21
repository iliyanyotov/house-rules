---
name: timeouts-everywhere
description: Use when writing or reviewing code that calls another system — HTTP, database, queue, RPC, LLM, third-party SDK. Use when adding a new external integration. Use when reviewing a function whose await chain has no timeout signal. Use when a deployment hangs, or a serverless function hits its hard wall.
---

# Timeouts, Everywhere

## Overview

**Every call that crosses a process boundary has an explicit per-call timeout.** No infinite waits, no implicit defaults, no "trust the SDK." If you can't name the timeout in seconds, you don't have one.

The acceptable values are documented numbers ("10s for DB reads, 5s for payment writes, 60s for LLM completions"). The unacceptable value is "whatever the library decides."

## The Iron Rule

```
NEVER make an external call without an explicit per-call deadline.
```

**No exceptions:**
- Not for "the library has a default"
- Not for "the platform will kill it anyway"
- Not for "the network is fast on internal calls"
- Not for "the LLM takes variable time"

## Why

A call without a timeout is a call that *might never return*. In a serverless world it manifests as a function hitting the platform's hard wall and dying with no context. In a long-running service, it manifests as exhausted connection pools, thread starvation, cascading failure.

The most common production outages aren't bugs — they're a downstream service being slow, and *the absence of timeouts* turning slowness into total unavailability. A 200ms timeout on a slow endpoint produces a fast failure that can be retried, fallen back from, or surfaced to the user. A missing timeout produces a hang that takes the whole system with it.

Timeouts are the single most leverage-per-line discipline in distributed systems. Every call. Every time.

## Detection

You are violating the rule if any of these are true:

- An `await fetch(...)` without `signal: AbortSignal.timeout(...)`.
- A database client initialized without a connection or statement timeout.
- An LLM call with no `signal` or `timeout` option.
- A third-party SDK constructed with no `timeout` parameter.
- A queue consumer's processing function with no overall deadline.
- The platform's hard wall is the *only* timeout (300s on serverless, 15min on Lambda) — that's a kill switch, not a timeout.
- A code review where "what happens if the dependency hangs?" has no clear answer.

## The Pattern

### `fetch` — always with `AbortSignal.timeout`

```ts
// ❌ Hangs forever if the server hangs.
const res = await fetch('https://api.example.com/data');

// ✅ Explicit deadline; AbortError on expiry.
const res = await fetch('https://api.example.com/data', {
  signal: AbortSignal.timeout(5_000),
});
```

`AbortSignal.timeout(ms)` is a built-in (Node 17+, all evergreen browsers). No library needed.

### Composing timeouts with user cancellation

```ts
// Combine a user-cancel signal with a timeout — abort if either fires.
const userCancel = new AbortController();
const signal = AbortSignal.any([userCancel.signal, AbortSignal.timeout(5_000)]);
await fetch(url, { signal });
```

### Database connections — set both connection and statement timeouts

```ts
// ✅ Connection-level timeout on the client.
const db = createDbClient({
  url: env.DATABASE_URL,
  connectTimeoutMs: 10_000,
});

// ✅ Statement-level timeout, set per session for long-running queries.
await db.execute(`SET statement_timeout = '15s'`);
```

Never leave `statement_timeout` at the server default — on most production databases that default is *unbounded*.

### Third-party SDKs — pass the timeout in the constructor

```ts
// Most SDKs accept a timeout in their constructor or per-call options.
// The specific option name varies; the discipline is the same.

// ✅ Payment provider SDK
const payments = new PaymentClient(env.PAYMENT_SECRET, {
  timeoutMs: 10_000,
  maxNetworkRetries: 0,   // control retries ourselves; see retry-with-budget
});

// ✅ Email provider SDK with per-call signal
await mailer.send({ to, subject, html }, {
  signal: AbortSignal.timeout(8_000),
});
```

Every SDK has a different surface. The rule: find it, set it, document the value.

### The injected-`fetch` seam — testable is not the same as timed

A common, *good* pattern: a client takes its `fetch` implementation as a constructor parameter so tests can inject a fake. The seam is excellent for testability — and does nothing for timeouts. These are two separate disciplines; the seam buys you the first, not the second.

```ts
// ❌ Good seam, no deadline. Every method routes through a naked call.
class SearchClient {
  constructor(
    private baseUrl: string,
    private fetchAPI: typeof fetch = fetch, // ← great for tests…
  ) {}

  // …but every one of these awaits with NO signal:
  async getUser(id: UserId): Promise<User> {
    const res = await this.fetchAPI(`${this.baseUrl}/users/${id}`);
    return res.json();
  }
  async searchOrders(q: string): Promise<Order[]> {
    const res = await this.fetchAPI(`${this.baseUrl}/orders?q=${q}`);
    return res.json();
  }
  // …and ~two dozen more, all the same shape.
}
```

**Blast radius.** The missing signal isn't one bug — it's one bug multiplied by every method. When dozens of calls all route through that single un-timed `this.fetchAPI(...)`, one slow or hung upstream pins a worker (or a pooled connection) indefinitely. Under load the pool drains, and now features that never touched `SearchClient` start stalling too. One missing line, many call sites, system-wide impact.

**The fix: add the deadline once, at the seam.** Wrap the injected `fetch` so every method inherits the timeout — no per-method edits, and the test seam is preserved.

```ts
// ✅ One wrapper at the seam; all methods inherit the deadline.
class SearchClient {
  private readonly call: typeof fetch;

  constructor(
    private baseUrl: string,
    fetchAPI: typeof fetch = fetch,
    private timeoutMs = TIMEOUTS_MS.http_external,
  ) {
    this.call = (input, init) =>
      fetchAPI(input, {
        ...init,
        // Combine any caller signal with our deadline.
        signal: init?.signal
          ? AbortSignal.any([init.signal, AbortSignal.timeout(this.timeoutMs)])
          : AbortSignal.timeout(this.timeoutMs),
      });
  }

  async getUser(id: UserId): Promise<User> {
    const res = await this.call(`${this.baseUrl}/users/${id}`);
    return res.json();
  }
  // searchOrders, and the other ~two dozen, now timed for free.
}
```

The injected `fetch` still lets tests pass a fake; the wrapper guarantees no method can ever issue a naked, deadline-free call. You need *both*: the seam for testing, the wrapper for timeouts.

### LLM calls — generous, but bounded

```ts
const result = await generateText({
  model,
  prompt,
  abortSignal: AbortSignal.timeout(60_000), // long, but bounded
});
```

LLMs can legitimately take 30–90s for long completions. The number is bigger; the discipline is the same.

### Outer timeout for multi-step operations

A function that does three external calls needs an outer timeout *in addition to* per-call timeouts:

```ts
async function fulfillOrder(orderId: OrderId): Promise<void> {
  const outer = AbortSignal.timeout(20_000);

  const inventory = await fetch(invUrl, {
    signal: AbortSignal.any([outer, AbortSignal.timeout(5_000)]),
  });
  const charge = await payments.charge({ orderId }, { signal: outer });
  await orders.markFulfilled(orderId);
}
```

If the inner timeouts each fire near their limit, the outer guards the *whole* deadline.

### Documenting timeout values — one source of truth

```ts
export const TIMEOUTS_MS = {
  db_query: 10_000,
  db_statement: 15_000,
  http_internal: 5_000,
  http_external: 10_000,
  payment_provider: 10_000,
  mailer: 8_000,
  llm: 60_000,
  webhook_dispatch: 5_000,
} as const;
```

Every call site references the same constant. Tuning is one edit.

## Pressure Resistance

### "The library has a default"

The library has a *secret* default that may be 30s, 60s, or unbounded. You don't know. The reader of your code doesn't know. The on-call engineer at 2am doesn't know. Make it explicit.

### "The platform will kill it anyway"

The platform's hard wall is a *kill switch*, not a timeout. It produces no graceful cleanup (DB transactions left open, locks held), no retriable error (a generic platform error), no fast failover to fallback paths, and bad observability. Your own timeout, set well below the platform's, produces a *named* error you control.

### "I don't know what value to use"

Start with a sensible default (5s for HTTP, 10s for DB, 60s for LLM). Watch the p99 latency in your error tracker / APM. Tune. Anything is better than nothing.

### "Timeouts cause flaky tests"

Flaky timeouts are a signal that either the value is too low or the dependency is slow. Both are useful signals. A *missing* timeout produces silent hangs — bugs you can't reproduce.

### "It's an internal call, the network is fast"

Internal calls hit databases, caches, queues, internal services — all with failure modes. Internal-network timeouts can be aggressive (200–2000ms) but they must exist.

## Red Flags

- `await fetch(...)` with no `signal`.
- A new SDK instance created with only the API key — no `timeout`, no `signal`.
- A `for await (...)` over a stream without a deadline.
- A `Promise.all` of network calls with no outer deadline.
- A function with multiple awaits where no one timeout is documented.
- A platform timeout showing up in logs as the *primary* failure mode.
- A flaky test fixed by *removing* the timeout — that's hiding the bug.

**All of these mean: a deadline is missing — name it in seconds, write it down.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The default is probably fine" | "Probably" is what bridges your code to a 3am page. |
| "It complicates the code" | One line per call. The complication is *reading* the system; the timeout reduces it. |
| "We'll add timeouts when we see problems" | The problem looks like "the whole service is down." Add them now. |
| "The library handles retries internally" | Retries are a separate discipline. Disable internal retries; control them yourself. |
| "Long-running jobs need long timeouts" | Yes — *named* long timeouts. Not no timeout. |
| "It's an idempotent read, hanging is fine" | Hanging is never fine. Pool exhaustion follows, eventually. |

## Reference

- Marc Brooker (AWS), [*Timeouts, retries, and backoff with jitter*](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/) — AWS Builders' Library. Required reading.
- Michael Nygard, *Release It!* 2e (2018), ch. 5 — introduces *Timeouts* as a foundational stability pattern, paired with circuit breakers and bulkheads.
