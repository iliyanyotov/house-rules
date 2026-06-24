---
name: bulkhead-isolated-failure-domains
description: Use when a single route, cron, or page calls multiple external dependencies (a payment provider + an email provider + an LLM + a cache) and they all share one timeout/retry/concurrency budget. Use when a slow third-party has caused unrelated features to degrade or time out. Use when reviewing code that uses one shared `withTimeout` helper or one shared HTTP client for every outbound call. Use when planning capacity for a serverless function that fans out to several APIs.
---

# Bulkhead: Isolated Failure Domains

## Overview

**Each external dependency gets its own failure domain** — its own timeout budget, its own concurrency cap, its own retry/breaker state. One slow third-party must not consume the budget every other dependency needs. One dep, one bulkhead. No shared pool.

The name comes from ship hulls: a watertight compartment that floods doesn't sink the ship because the next compartment's wall holds. Same rule here — if the email provider is slow, the bulkhead around it fills up; the bulkhead around the database stays dry.

## The Iron Rule

```
NEVER share a timeout, concurrency cap, or breaker across dependencies. One bulkhead per dep.
```

**No exceptions:**
- Not for "all our deps are fast"
- Not for "it's the same `withTimeout` either way"
- Not for "it's overkill for a small app"
- Not for "concurrency caps will reject good requests"

## Why

Timeouts and circuit-breakers are *per-call* disciplines. Bulkheads are *per-dependency-class* disciplines — one layer outward.

Consider a route that does four outbound things:

```
POST /api/checkout
 ├── payments.charge()      (payment provider)    — critical path
 ├── db.insert(orders)      (database)            — critical path
 ├── bot.verify()           (bot-detection)       — degradable (fail-open or skip)
 └── mailer.send()          (email provider)      — degradable (fire-and-forget)
```

If all four share *one* `withTimeout(15_000)` helper and *one* implicit concurrency model ("one function instance at a time"), then a 14-second mailer hang blocks the function instance for 14 seconds. Payments, bot-detection, and the database are fine — but they wait behind the mailer because the instance is busy. Multiply by traffic: your `/api/checkout` p99 latency *is* the mailer's p99 latency, even on requests where the email never matters.

The bulkhead rule: give each dependency its **own** timeout (sized to that dep's real p99), its **own** in-flight cap, its **own** breaker state. When the mailer hangs, only the mailer bulkhead fills. The other three deps have free capacity.

This converts "a third-party outage takes down /api/checkout" into "a third-party outage degrades the email step alone — the order still books."

## Detection

You are violating the rule if any of these are true:

- One `withTimeout` helper or one shared HTTP client wraps calls to multiple unrelated dependencies.
- A single `TIMEOUTS_MS` constant is used for every outbound call regardless of dependency.
- A single breaker (or no breaker) covers calls to multiple dependencies.
- A function's p99 latency tracks the *slowest* dependency it ever calls — even on requests where that dep is incidental.
- A code review can't answer "if dep X is at 20s p99, what happens to a request that only needs dep Y?"
- An incident postmortem identifies one slow dep, but the user-visible degradation was across *all* features the function exposes.
- A `Promise.all([...])` (or `deps.map((d) => callDep(d))`) over calls to *different* dependency classes, sharing one try-catch and no per-dep timeout — a hang in any element blocks settlement of all.

## The Pattern

### Per-dependency timeout and in-flight cap

```ts
// One bulkhead per dependency. Each owns:
//   - timeout sized to that dep's real p99 + headroom
//   - in-flight cap so a hang can't eat the whole function instance
//   - (optional) breaker that lives inside the bulkhead

type Bulkhead = {
  run: <R>(op: (signal: AbortSignal) => Promise<R>) => Promise<R>;
};

class BulkheadSaturatedError extends Error {
  constructor(name: string) { super(`${name}_bulkhead_saturated`); }
}

function makeBulkhead(name: string, opts: { timeoutMs: number; maxInFlight: number }): Bulkhead {
  let inFlight = 0;
  const breaker = new Breaker(name, { /* see circuit-breaker-on-flaky-deps */ });

  return {
    async run(op) {
      if (inFlight >= opts.maxInFlight) {
        throw new BulkheadSaturatedError(name);
      }
      inFlight++;
      try {
        return await breaker.run(
          () => op(AbortSignal.timeout(opts.timeoutMs)),
          () => { throw new Error(`${name}_unavailable`); },
        );
      } finally {
        inFlight--;
      }
    },
  };
}

// Each dep's bulkhead is sized for that dep, not the route.
export const paymentBulkhead = makeBulkhead('payment', { timeoutMs: 10_000, maxInFlight: 50 });
export const mailerBulkhead = makeBulkhead('mailer', { timeoutMs: 3_000, maxInFlight: 20 });
export const botBulkhead = makeBulkhead('bot', { timeoutMs: 2_000, maxInFlight: 50 });
export const dbBulkhead = makeBulkhead('db', { timeoutMs: 5_000, maxInFlight: 100 });
```

### Using it in a handler

```ts
export async function handleCheckout(req: Request) {
  const parsed = CheckoutInput.parse(await req.json());

  // Each step runs inside its own bulkhead.
  // A hang in any one step can't drain the others' capacity.
  const verified = await botBulkhead.run((signal) =>
    bot.verify(parsed.token, { signal })
  );
  if (!verified) return json({ error: 'bot' }, 400);

  const charge = await paymentBulkhead.run((signal) =>
    payments.charge(parsed.charge, { signal })
  );
  const order = await dbBulkhead.run((signal) =>
    db.insert(orders).values({ chargeId: charge.id }).returning() // Drizzle on Postgres
  );

  // Email is non-critical: degrade gracefully, don't fail the request.
  mailerBulkhead.run((signal) =>
    mailer.send(welcomeEmail(order), { signal })
  ).catch((err) =>
    log.warn('welcome_email_failed', { orderId: order.id, err: err.message })
  );

  return json({ order });
}
```

The shape that most often *lacks* bulkheads is a parallel fan-out — `Promise.all` over several deps with one shared catch and no per-dep budget:

```ts
// ❌ One hung dep delays settlement of all; one rejection loses the others' results.
const [user, orders, recs] = await Promise.all([fetchUser(id), fetchOrders(id), fetchRecs(id)]);

// ✅ Each call runs in its own bulkhead (own timeout + cap); allSettled keeps failures isolated.
const [user, orders, recs] = await Promise.allSettled([
  userBulkhead.run((s) => fetchUser(id, { signal: s })),
  orderBulkhead.run((s) => fetchOrders(id, { signal: s })),
  recsBulkhead.run((s) => fetchRecs(id, { signal: s })),
]);
```

Read top-to-bottom, each call is annotated by its bulkhead. A reader instantly knows: which dep does this talk to, what budget does it get, what's the blast radius if it hangs.

### Sizing each bulkhead

Three knobs per dep, set from real data — not guesses:

| Knob | How to set it |
|---|---|
| `timeoutMs` | The dep's documented p99 × 1.5, capped by your own user-experience budget. A fast API gets 2-3s; an LLM gets 60s. |
| `maxInFlight` | What share of your function instance is acceptable to lose to this dep? A non-critical dep (email) gets a *small* cap so its saturation can't displace critical traffic. A critical dep (payments) gets a larger one. |
| Breaker thresholds | Different deps trip at different rates. Stable internal deps rarely trip; experimental third-parties trip often. |

The numbers don't have to be perfect on day one. They have to *exist* per dep.

### What goes inside a bulkhead, what doesn't

Inside: any call that crosses a process boundary (HTTP, DB, queue, cache, file storage).

Outside: pure functions, in-memory work, request-scoped state. These can't hang the way a network call can; wrapping them in a bulkhead is noise.

### Composition with timeouts, retries, and breakers

The layering, outermost to innermost:

```
[Bulkhead]              ← per-dependency in-flight cap; fails fast if saturated
  └── [Breaker]         ← per-dependency state machine; opens on sustained failure
        └── [Retry]     ← per-call retry budget with jitter
              └── [Timeout per attempt]      ← per-attempt abort signal
```

Each layer guards against a different failure mode. Bulkheads guard against *capacity exhaustion*; breakers against *sustained outage*; retries against *transient flakes*; timeouts against *individual slowness*.

### One bulkhead per dep, not per call site

The bulkhead's state (in-flight count, breaker state) is *shared* across every call site that talks to that dep. Define it once at module scope and import everywhere — never `new Bulkhead()` inside a request handler.

## Pressure Resistance

### "All our deps are fast — we don't need bulkheads"

Until one isn't. The cost of adding bulkheads up front is one config-line per dep; the cost of retrofitting them during an incident is paid in customer-visible degradation. Build the muscle while everything's calm.

### "It's the same `withTimeout` either way"

It is *not* the same `withTimeout`. A shared helper with one budget means a slow dep eats the budget that a fast dep needed. Per-dep timeouts mean each gets exactly the budget that matches its real p99.

### "Concurrency caps will throw spurious errors under load"

That's the design. A saturated bulkhead returns a *named* failure — `${dep}_bulkhead_saturated` — fast, instead of letting the function instance hang. The caller can degrade or fail the request cleanly. Compare to the alternative: hang for 30s and then 503 anyway.

### "It's overkill for a small app"

The pattern scales down. A route calling three APIs is the minimum size where bulkheads earn their weight. If a route calls *one* dep, it doesn't need a bulkhead — just a timeout. The threshold is: ≥2 deps in one critical path.

### "Bulkheads complicate the call sites"

Look at the example above. Each call is *more* readable, not less — every outbound call is now visibly labeled with the dep it talks to. The clarity gain offsets the structural cost.

### "How do I know the right `maxInFlight`?"

Start permissive: half your function-instance concurrency limit per critical dep, one-quarter for non-critical. Tune from real incident data. The *exact* number matters less than *having one*.

## Red Flags

- A `TIMEOUTS_MS = { default: 5000 }` constant used everywhere.
- One `withTimeout(promise, 5000)` helper applied to every outbound call regardless of dep.
- A code review that says "we already have timeouts" — without distinguishing per-dep budgets.
- An incident report where one slow dep degraded *all* features of a route.
- A function instance whose p99 tracks the slowest dep, even on requests that barely touch it.
- A test that mocks four deps with one shared mock — the production code likely shares one bulkhead too.
- A `Promise.all([...])` fanning out to several different deps with no per-element timeout — the slowest sets the wall-clock and one hang blocks the whole batch.

**All of these mean: the failure domains aren't isolated — one slow dep will take everything else with it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "We have timeouts already" | Per-call timeouts ≠ per-dep capacity isolation. Both layers are needed. |
| "All our deps are fast" | "Are" ≠ "will always be." Bulkheads pay off in the first incident. |
| "It's complex" | It's a config object per dep. Less complex than the incident postmortem. |
| "We don't have that many requests" | At low traffic, bulkheads cost nothing. At high traffic, they save everything. |
| "I'll add it when there's a problem" | The problem is invisible until it isn't. Once the dep starts hanging, you've already shipped the bug. |
| "Concurrency caps will reject good requests" | They reject requests that would have *hung anyway*. The user sees a fast 503 instead of a 30s spinner. |

## Related

- `circuit-breaker-on-flaky-deps` — layered: a breaker per dependency, inside the bulkhead
- `timeouts-everywhere` — a per-dependency timeout is part of the design
- `shed-load-under-overload` — bulkhead saturation is the shed signal

## Reference

- Michael Nygard, *Release It!* 2e (2018), ch. 5 — names "Bulkheads" alongside Timeouts and Circuit Breaker as core stability patterns. The metaphor is from naval architecture.
- Sam Newman, *Building Microservices* 2e (2021) — applies the same pattern at the service level (a slow downstream service doesn't drain the calling service's connection pool).
- Netflix Hystrix design notes — the production reference. Hystrix is in maintenance, but the pattern survived in `resilience4j`, `cockatiel`, `opossum`, and the JS ecosystem's hand-rolled equivalents.
