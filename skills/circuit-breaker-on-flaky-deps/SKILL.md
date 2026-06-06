---
name: circuit-breaker-on-flaky-deps
description: Use when integrating with an external dependency that has a non-trivial failure rate — payment processors, LLM APIs, third-party search, partner services, internal services with known intermittency. Use when a slow dependency is degrading unrelated parts of the system. Use when log dashboards show a flat-top of timeouts during an upstream outage.
---

# Circuit Breaker on Flaky Dependencies

## Overview

**Any external dependency with non-trivial failure rate is called through a circuit breaker** with explicit open / half-open thresholds. When the breaker is open, the system fails fast with a documented fallback — it does not wait for the dependency.

The breaker is a per-dependency state machine. Tripping it is automatic. Recovering from it is deliberate.

## The Iron Rule

```
NEVER call a flaky dependency directly. Wrap it in a breaker with thresholds, cooldowns, and a named fallback.
```

**No exceptions:**
- Not for "we don't have that many failures"
- Not for "retries are enough"
- Not for "adding a breaker is complex"
- Not for "I'll add it when I see problems"

## Why

Timeouts and retries handle *single* slow or failing calls. Circuit breakers handle the case when the dependency is *broken for a sustained period*.

Without a breaker, every call to the broken dependency:
1. Waits for its timeout (e.g. 5s).
2. Maybe retries (× 2 or 3).
3. Holds a connection, a function instance, a request slot.
4. Eventually fails.

Multiply by traffic. At even modest scale, 5-second timeouts × continuous traffic = exhausted connection pools, blocked workers, cascading failure of *unrelated* features that share infrastructure. One dependency's outage becomes your full-system outage.

The circuit breaker breaks that loop: after enough failures, stop calling the dependency at all. Return the fallback immediately. Periodically test if it's recovered. Resume only when it is.

This converts "the slow dependency takes us down" into "the slow dependency is gracefully unavailable."

## Detection

You are violating the rule if any of these are true:

- A dependency with a history of brief outages is called directly on every request.
- A timeout dashboard shows a flat-topped error rate during upstream incidents.
- User-facing latency spikes during a third-party incident because every request still waits the timeout.
- A function instance hits its limit during a payment-processor outage and unrelated features go down.
- An LLM call is in the critical path of a page render with no breaker.
- A code review can't answer "what happens if the dependency is down for 10 minutes?"

## The Pattern

### The three states

```
   [CLOSED] ── too many failures ──→ [OPEN]
       ↑                              │
       │                              │  cooldown period
       │                              ↓
       └── successful probe ── [HALF-OPEN]
                                      │
                                      └── failed probe ──→ [OPEN]
```

- **CLOSED:** normal. Requests flow. Failures are counted.
- **OPEN:** breaker is tripped. Calls fail fast (no actual request). Returns fallback.
- **HALF-OPEN:** after a cooldown, allow one or a few requests through. If they succeed, return to CLOSED. If they fail, back to OPEN.

### Minimal implementation

```ts
type BreakerState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

type BreakerOptions = {
  failureThreshold: number;          // failures within window to trip
  windowMs: number;                  // sliding window for failure counting
  cooldownMs: number;                // time in OPEN before probing
  successThresholdHalfOpen: number;  // successes needed to close
};

export class Breaker {
  private state: BreakerState = 'CLOSED';
  private failures: number[] = [];
  private halfOpenSuccesses = 0;
  private openedAt = 0;

  constructor(private name: string, private opts: BreakerOptions) {}

  async run<T>(op: () => Promise<T>, fallback: () => Promise<T> | T): Promise<T> {
    this.transitionIfDue();

    if (this.state === 'OPEN') {
      return fallback();
    }

    try {
      const result = await op();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      if (this.state === 'OPEN') return fallback();
      throw err;
    }
  }

  private transitionIfDue() {
    if (this.state === 'OPEN' && Date.now() - this.openedAt >= this.opts.cooldownMs) {
      this.state = 'HALF_OPEN';
      this.halfOpenSuccesses = 0;
    }
  }

  private onSuccess() {
    if (this.state === 'HALF_OPEN') {
      this.halfOpenSuccesses++;
      if (this.halfOpenSuccesses >= this.opts.successThresholdHalfOpen) {
        this.state = 'CLOSED';
        this.failures = [];
      }
    }
  }

  private onFailure() {
    const now = Date.now();
    this.failures.push(now);
    this.failures = this.failures.filter(t => now - t < this.opts.windowMs);
    if (this.state === 'HALF_OPEN') {
      this.trip();
    } else if (this.state === 'CLOSED' && this.failures.length >= this.opts.failureThreshold) {
      this.trip();
    }
  }

  private trip() {
    this.state = 'OPEN';
    this.openedAt = Date.now();
    counter('breaker.tripped', 1, { dependency: this.name });
  }
}
```

Production libraries add observability, half-open concurrency limits, and per-error-type filtering — but the state machine above is the minimum.

### Using it

```ts
const breaker = new Breaker('payment-provider', {
  failureThreshold: 5,         // 5 failures in 30s trips
  windowMs: 30_000,
  cooldownMs: 30_000,          // stay OPEN for 30s
  successThresholdHalfOpen: 2, // 2 consecutive successes to close
});

export async function chargeWithBreaker(amountCents: number, source: string) {
  return breaker.run(
    () => payments.charge({ amountCents, source }),
    () => { throw new Error('payment_unavailable'); }, // fallback — fail fast
  );
}
```

The fallback is dependency-specific:
- **Payment:** throw a typed `payment_unavailable` error → UI shows "try again."
- **Search:** return cached or default results.
- **Email send:** queue for later (a queue is a fallback).
- **LLM:** return a templated/cached response, or skip the feature gracefully.

### Per-dependency state, not per-request

The breaker is *shared* across all calls to that dependency. One breaker per provider, used by every call. The `failures` array must be observable across requests — in serverless runtimes, this means module-scope state (shared across requests on the same instance). In a horizontally-scaled service, per-instance breakers are usually fine; for stricter coordination, use a shared state store.

### What "non-trivial failure rate" means

Not every dependency needs a breaker:

| Dependency | Breaker? |
|---|---|
| Internal database with healthy SLA | Probably not — fail loud and rely on alerts. |
| Payment provider, email provider, LLM provider | Yes — known intermittent failures. |
| First-party microservices with their own SLOs | Yes — saves cascading outages. |
| Cache (single-region) | Yes — cache misses fall back to source. |
| In-process operations | No — there's no "external dependency" to break from. |

Rule of thumb: if you've ever seen its dashboard go red, it needs a breaker.

### Composition with timeouts and retries

```
[Caller]
   │
   └── Breaker.run()                  ← if OPEN, fallback immediately
         │
         └── withRetry()              ← per-call retry policy
               │
               └── fetch(..., {signal: timeout})  ← per-attempt timeout
```

The breaker counts *failures after retries*. If the retry succeeded, the breaker sees success. If retries exhausted, the breaker sees one failure (not three).

## Pressure Resistance

### "We don't have that many failures"

You don't have many *yet*. Breakers are insurance — cheap to add, life-saving when the dependency does fail. The dependency *will* fail; the only question is how the system handles it.

### "Retries are enough"

Retries handle the *first* slow request. They don't help when the dependency is down for minutes — every fresh request still pays the full timeout × retry budget. The breaker recognizes the pattern and stops paying.

### "Adding a breaker is complex"

The minimal implementation is ~50 lines. Production libraries are two imports. The complexity is one-time; the benefit is every future incident.

### "I'll add it when I see problems"

You'll see them as user-visible total outages. The window to add is *before* — during calm. Build the muscle now.

### "Fallbacks are dishonest — users should know it failed"

Fallbacks are *named* failures. The UI shows "search unavailable, showing cached results" — not silent corruption. Honest fallbacks beat dishonest timeouts (where the user just sees a hang).

### "We don't have a fallback"

Then the fallback is "fail fast with a typed error." The breaker still helps — it stops the system from queuing retries while the dependency is down. The user sees an error in 100ms instead of 30s.

## Red Flags

- A dependency with documented intermittency called directly with no breaker.
- A timeout-error dashboard with a flat-topped error rate during upstream incidents.
- A function-instance saturation alert that traces to a single slow dependency.
- A code review where "what's the fallback when X is down?" has no answer.
- A breaker library imported but configured with `failureThreshold: 1000` — effectively disabled.
- A breaker that never trips in production (failures *do* happen — your threshold is wrong, or the breaker isn't wired).

**All of these mean: the breaker is missing, mis-tuned, or being bypassed — fix it before the next outage.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Retries handle it" | Retries don't handle sustained outages. They aggravate them. |
| "Failures are rare" | Until they aren't. Breakers cost almost nothing in the rare-failure case. |
| "It's a single instance, no need for distributed state" | Per-instance state is fine for most cases. Don't let "distributed perfection" block "single-instance good." |
| "The fallback is hard to design" | Even "throw immediately" is a valid fallback — better than waiting the timeout. |
| "It complicates traces" | Tag the trace with breaker state. Now traces have *more* signal. |
| "Library is overkill, I'll add it later" | Build the minimal version (above). Replace with library when you need observability/concurrency knobs. |

## Reference

- Michael Nygard, *Release It!* 2e (2018), ch. 5 — the canonical chapter on stability patterns (timeouts, circuit breakers, bulkheads, handshaking, steady-state).
- Netflix Hystrix design notes (archived) — the production-grade reference. Hystrix itself is in maintenance mode, but the pattern survived; modern libraries (`opossum`, `cockatiel`, `resilience4j`) implement it.
- Martin Fowler, ["CircuitBreaker"](https://martinfowler.com/bliki/CircuitBreaker.html) — the canonical short essay describing the state machine.
