---
name: shed-load-under-overload
description: Use when designing an API endpoint that could receive traffic spikes (signup form during a launch, contact form during a campaign, cron-triggered fan-out, viral page going to HN). Use when reviewing a route that queues work without bound during a downstream slowdown. Use when an incident postmortem says "the database got overloaded and the function instances kept piling up." Use when a queue or in-memory buffer holds more than a few seconds of work. Use when "we should rate-limit this" comes up but no concrete saturation signal is defined.
---

# Shed Load Under Overload

## Overview

**Under load that the downstream can't keep up with, reject incoming work *fast* with a 429 or 503** (and an honest `Retry-After`) — do not queue indefinitely. The decision to shed is automatic and based on an observable saturation signal: in-flight count, queue depth, p99 latency, or upstream-error rate.

Once shedding, return *immediately*; do not pay the cost of any downstream call you're about to fail anyway.

## The Iron Rule

```
NEVER queue indefinitely under overload. Shed fast with 503 + Retry-After.
```

**No exceptions:**
- Not for "we don't get enough traffic to overload"
- Not for "the platform auto-scales"
- Not for "returning 503 looks bad to users"
- Not for "we'll lose orders during the spike"

## Why

When load exceeds capacity, the system has three options:

1. **Queue everything.** Each new request waits behind the backlog. Queue depth grows; per-request latency grows; eventually function instances time out, retries pile on, downstream dependencies stay saturated long after the original spike ended. The system enters a death spiral where it's *less* available than if it had simply refused work.

2. **Drop work randomly.** Better than queueing, but it discards work indiscriminately — including work from clients with no retry logic, or work that's been waiting longest.

3. **Shed at the entry point.** Detect saturation; reject *new* work *fast* with a clear signal (HTTP 503 + `Retry-After`); let work already in flight finish.

Option 3 is *shedding load*. It's the only option that lets the system *catch up* — every shed request frees a slot for a request already mid-flight, and the backlog drains. Options 1 and 2 keep the system pinned at saturation indefinitely.

The honest 503 + `Retry-After` is a *contract with the caller*. Modern HTTP clients, retry libraries, and most SDKs honor `Retry-After`. Sending one tells the caller "wait this many seconds before retrying" and they distribute their retries — instead of all hammering you the next second.

## Detection

You are violating the rule if any of these are true:

- A route handler has no saturation check and unconditionally processes every request to completion.
- A queue (in-memory or external) grows unbounded under spike traffic; no eviction or drop-oldest policy.
- A function with a 60s `maxDuration` calls a downstream that sometimes takes 30+ seconds — the function has been pulling requests deep into its concurrency budget.
- An incident postmortem says "we were getting 503s from the downstream and our requests kept piling up."
- A retry strategy on a client honors no `Retry-After` (or the server never sends one).
- A "rate limit" is described in the design but no concrete trigger (request count? queue depth? latency?) is specified.
- A bulkhead is configured with `maxInFlight: 1000` — effectively no shedding for any realistic single-region traffic.

## The Pattern

### Detect saturation; reject fast; tell the caller when to retry

```ts
// ✅ Bulkhead with a clear shed signal (see bulkhead-isolated-failure-domains).
const checkoutBulkhead = makeBulkhead('checkout', {
  timeoutMs: 10_000,
  maxInFlight: 50,           // shed when 51st request arrives
});

export async function handleCheckout(req: Request) {
  try {
    const order = await checkoutBulkhead.run((signal) => placeOrder(req, signal));
    return json({ order });
  } catch (err) {
    if (err instanceof BulkheadSaturatedError) {
      return new Response(JSON.stringify({ error: 'overloaded' }), {
        status: 503,
        headers: {
          'Retry-After': '10',
          'Content-Type': 'application/json',
        },
      });
    }
    throw err;
  }
}
```

The 503 returns *immediately* — no DB call, no third-party hit, no logging that itself costs latency. The `Retry-After: 10` tells the client "back off for 10 seconds." A polite client will. An impolite one is going to hammer you regardless — the 503 still costs ~zero to send, so you survive either way.

### Define the saturation signal explicitly

```ts
type SaturationState = 'healthy' | 'degraded' | 'overloaded';

function checkSaturation(): SaturationState {
  const recentP99Ms = getRecentP99Ms();
  const inFlight = getCurrentInFlight();

  // Thresholds set from real baseline; tune from real incidents.
  if (inFlight > 100 || recentP99Ms > 5_000) return 'overloaded';
  if (inFlight > 50  || recentP99Ms > 2_000) return 'degraded';
  return 'healthy';
}
```

Three states because the response differs:

- **healthy:** serve normally.
- **degraded:** serve, but skip non-critical work (analytics ping, welcome email).
- **overloaded:** shed with 503.

The thresholds aren't aspirational; they're observed from your own data. Pick them initially as a guess, *write them down*, and tighten/loosen from real incidents.

### `Retry-After` — honest and bounded

```ts
// ✅ Honest Retry-After
const retryAfter = Math.min(60, Math.ceil(estimatedRecoverySeconds()));
return new Response('overloaded', {
  status: 503,
  headers: { 'Retry-After': String(retryAfter) },
});

// ❌ Lying Retry-After
return new Response('overloaded', {
  status: 503,
  headers: { 'Retry-After': '1' },
});
// Sending Retry-After: 1 to 10k clients means 10k retries one second later — re-spike.
```

`Retry-After` is *added jitter* if the caller respects it. Underestimate and you re-spike; overestimate and you delay healthy traffic. Calibrate from real recovery time. When in doubt, prefer the larger value.

`Retry-After` is the most direct carrier, but it's not the only honest one. The `RateLimit-*` header family (IETF `RateLimit-Limit` / `RateLimit-Remaining` / `RateLimit-Reset`, or the older `X-RateLimit-*`) carries the same recovery-time information and is what many rate limiters emit instead. Either is honest as long as the value reflects *real* recovery time; a 429/503 whose retry timing is buried only in the JSON body (with no header at all) is the form to avoid — a machine can't read it.

### Different shedding for different criticality

```ts
// ✅ Critical: shed only at hard saturation.
//    Non-critical: shed earlier so critical paths have headroom.
export async function handlePost(req: Request) {
  const isAnalytics = req.headers.get('x-purpose') === 'analytics';
  const saturation = checkSaturation();

  if (saturation === 'overloaded') return overloadedResponse();
  if (saturation === 'degraded' && isAnalytics) return overloadedResponse(); // shed non-critical first

  // ... handle ...
}
```

This is shed-load in service of priority. Critical traffic gets all available capacity; non-critical sheds early. The categorization matters more than the exact threshold.

### In-process queues — bounded with drop policy

```ts
// ❌ Unbounded queue accumulates work forever.
const pending: WorkItem[] = [];
function enqueue(item: WorkItem) {
  pending.push(item);
}

// ✅ Bounded queue with drop-oldest policy.
const MAX_QUEUE = 1000;
const pending: WorkItem[] = [];
function enqueue(item: WorkItem): 'queued' | 'shed' {
  if (pending.length >= MAX_QUEUE) {
    return 'shed';  // caller decides: 503 / fallback / drop / log
  }
  pending.push(item);
  return 'queued';
}
```

The bound is *whatever your worker can drain in a reasonable time*. If a worker drains 100/s and `Retry-After` is 10s, the queue should hold ~1000. Beyond that, you're queueing more than you'll process.

### Serverless platforms shed automatically — but slowly

Serverless function instances have concurrency limits. The platform sheds *automatically* — but the failure mode is a 30s wait followed by a `FUNCTION_INVOCATION_TIMEOUT` or similar. To shed *faster*, put the saturation check inside the route handler so the 503 returns in milliseconds, not seconds:

```ts
// ✅ Shed inside the handler, before paying any downstream cost.
export async function handlePost(req: Request) {
  if (checkSaturation() === 'overloaded') return overloadedResponse();
  // ... rest of handler
}
```

### When *not* to shed — internal/admin endpoints

```ts
// ✅ Internal admin endpoint — fail loud, don't shed; it's a single operator request.
export async function handleAdminMaintenance(req: Request) {
  await authorizeAdmin(req);
  // No shed check; if the system is overloaded, the operator needs to know,
  // not get a 503 they'll just retry.
  await runMaintenanceTask();
  return json({ ok: true });
}
```

Shedding is a *public-traffic* discipline. Admin paths, health checks, and internal probes should not shed — they're how the operator sees the overload.

## Pressure Resistance

### "We don't get enough traffic to overload"

Until you do. The first time is *also* the time you're not paying attention. The Hacker News spike, the marketing campaign that worked, the cron-triggered fan-out that misfired — all real cases that have taken systems down. Shedding costs nothing in the no-overload case; it saves everything in the overload case.

### "The platform auto-scales — we don't need to shed"

The platform scales *function instances*. Function instances don't help when the bottleneck is your *downstream* — the database's connection pool, a payment provider's per-account rate limit, an email provider's quota. The scaling makes the bottleneck *worse* (more instances all hitting the saturated downstream simultaneously). Shedding at the route is the discipline that protects the downstream.

### "Returning 503 looks bad to users"

A 503 with `Retry-After: 10` and a clear error page looks much better than a 30-second hang followed by a generic timeout. Users *understand* "try again in a moment." They don't understand "the spinner has been spinning for 28 seconds."

### "We'll lose orders / signups during the spike"

You lose them anyway. Either:
- (a) you shed → some lose, some get through fast, system recovers, you can serve the retries cleanly; or
- (b) you queue → almost all lose, system stays saturated, retries pile on the saturation, downstream stays broken longer.

Option (a) loses *fewer* orders in total. Option (b) is the death spiral.

### "We should add a queue / buffer instead"

Queues and buffers don't *solve* overload — they *delay* it. A buffer that fills under spike traffic still has to drain afterwards, and now the *post-spike* traffic is queued behind a buffer of stale spike traffic. Unless the buffer has a bounded size + drop policy, you've just moved the problem.

### "Retry-After is just a hint"

Modern clients (most SDKs, AWS SDK, browser `fetch` retry libraries) honor `Retry-After`, and many also read the `RateLimit-Reset` family. Even crude clients benefit because *some* portion of traffic backs off, which is enough to break the death spiral.

## Red Flags

- A route handler that does *zero* checks before calling its downstream.
- A queue with no max size.
- A function whose downstream sometimes takes 30+ seconds with no shed check.
- An incident postmortem ending in "we manually killed the function instances."
- A `Retry-After: 0` or `Retry-After: 1` sent during real overload.
- A "rate-limited" message returned only when manually toggled by an operator.
- A bulkhead with `maxInFlight` larger than the platform concurrency × 10.
- The phrase "we'll just queue them and process later" with no bounds on "later."

**All of these mean: the system has no defined behavior for overload — it will collapse instead of degrading.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Auto-scaling handles it" | Auto-scaling scales the *handler*, not the downstream. The bottleneck moves; it doesn't disappear. |
| "We can't predict overload" | You don't have to. The saturation signal tells you when it's happening. Set thresholds; tune from incidents. |
| "Queueing is more user-friendly" | Queueing is *less* user-friendly under real overload. Honest 503s beat slow timeouts. |
| "We don't have spike traffic" | You don't have spike traffic *yet*. The first time happens before you notice it. |
| "Retry-After is ignored" | Not by SDKs and modern clients. Even partial compliance breaks the death spiral. |
| "We have rate limiting" | Rate limiting is "per-client cap." Shedding is "system-wide capacity." They're complementary; you need both. |
| "Queue + workers can keep up" | At sustained overload, no they can't. Add a bound. |

## Related

- `bulkhead-isolated-failure-domains` — saturation is the signal to shed (503)
- `graceful-degradation-defaults` — same posture: degrade rather than queue forever

## Reference

- Michael Nygard, *Release It!* 2e (2018), ch. 5 — names "Shed Load" and "Back Pressure" as paired stability patterns.
- Google SRE Book, ch. 21 ("Handling Overload") — *"It is essential that overload behavior be a first-class concern in service design."*
- AWS Builders' Library, [*Using load shedding to avoid overload*](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/) — production case study covering `Retry-After`, priority-based shedding, and saturation signals.
