---
name: health-and-readiness-endpoints
description: Use when a service's only health endpoint is an unconditional 200 stub, or when one `/health` route serves both the restart decision and the routing decision. Use when a liveness probe checks the database and a DB blip restarts the whole fleet. Use when a health check fans out to every dependency on every probe and becomes its own load source. Use when traffic keeps routing to an instance whose DB pool is exhausted, or when a partner-API outage marks every replica NotReady at once.
---

# Health and Readiness Endpoints

## Overview

**A long-running service exposes two different answers to two different questions: liveness ("is this process wedged?" — process-local, never checks a dependency) and readiness ("should traffic route here right now?" — critical dependencies usable, not draining).** They exist because the orchestrator reacts differently: a failed liveness probe restarts the instance; a failed readiness probe only stops routing to it.

Conflate them, or stub them, and the probes act on wrong information — which means restarts that fix nothing and traffic sent to instances that can't serve it.

## The Default Rule

```
Two endpoints. Liveness: process-local only — a dependency check in liveness
is a restart storm waiting for a blip. Readiness: critical deps + draining
state, answered from a cached background check — never a live fan-out per probe.
```

This is a default, not an invariant — the named exceptions:

- **Serverless/edge functions**: the platform owns instance lifecycle and routing; there is no probe to answer.
- **One-shot jobs and CLIs**: exit codes are their health protocol.
- **A platform offering only one probe**: serve it the *readiness* semantics (routing decision), and leave restarts to crash-on-fatal — restarting on dependency failure is the harmful half.

Within the default, one clause is effectively non-negotiable: liveness never checks a dependency. Restarting your process has never fixed someone else's outage.

## Why

Three distinct failure modes, one per wrong shape:

- **The shallow stub.** `/health` returns 200 unconditionally. An instance whose DB pool is exhausted, whose config failed to load, or which is mid-shutdown reports healthy, and the load balancer keeps feeding it users. The check exists, so dashboards are green; every request into that instance fails. False confidence is worse than no check — someone would have noticed no check.
- **The over-deep liveness.** Liveness pings the database. The database blips for 40 seconds; every replica's liveness fails; the orchestrator restarts the entire fleet at once. The restarts don't fix the DB — and now you have a cold-start stampede *on top of* the DB blip. A dependency outage was amplified into a self-inflicted platform outage.
- **The deep-check cascade.** Readiness fans out to every dependency, live, per probe: N replicas × M dependencies × a probe every few seconds = a permanent synthetic load source that scales with your fleet. The health check DDoSes the database, trips partner rate limits — and if downstream services' health checks call *their* dependencies' health checks, one slow leaf makes whole chains flap in sympathy.

## Detection

You are violating the rule if any of these are true:

- One `/health` endpoint wired to both the restart probe and the routing probe.
- A liveness handler that touches the DB, cache, queue, or any network dependency.
- A readiness handler that returns a hard-coded 200.
- A readiness handler that awaits live dependency calls per probe instead of reading a cached background check.
- Readiness gating on a *non-critical* dependency — a recommendations-provider outage marking every replica NotReady simultaneously.
- No "draining" state — instances shut down while still advertising ready.
- Dependency-check calls with no timeout, so a slow dependency makes the probe itself hang past the probe's own deadline.

## The Pattern

### Two questions, two endpoints

| | Liveness | Readiness |
|---|---|---|
| Question | "Is the process wedged?" | "Route traffic here right now?" |
| Checks | Process-local only — never a dependency | Critical deps usable + not draining |
| On failure | Orchestrator **restarts** the instance | Orchestrator **stops routing**; process keeps running |
| Wrong-answer cost | Restart storms during dependency blips | Traffic to broken instances, or fleet-wide self-eviction |

```ts
export function handleLiveness(): Response {
  // Answering at all is the check: the process is up and the event loop turns.
  return json({ status: 'alive' }, { status: 200 });
}
```

### Readiness reads a cache; background loops do the checking

The probe must be cheap and constant-cost no matter how often it fires. Dependencies are checked on a fixed interval by background loops; the endpoint reads their last result:

```ts
type DependencyName = 'db' | 'queue';
const CRITICAL_DEPS: readonly DependencyName[] = ['db', 'queue'];

type CheckResult = { ok: boolean; checkedAt: number };
const lastCheck = new Map<DependencyName, CheckResult>();

export function startReadinessChecks(): void {
  void checkForever('db', () => pingDb({ timeoutMs: 2_000 }));      // e.g. SELECT 1 on the pool
  void checkForever('queue', () => pingQueue({ timeoutMs: 2_000 }));
}

async function checkForever(dep: DependencyName, ping: () => Promise<void>): Promise<never> {
  for (;;) {
    try {
      await ping();
      lastCheck.set(dep, { ok: true, checkedAt: Date.now() });
    } catch (err) {
      lastCheck.set(dep, { ok: false, checkedAt: Date.now() });
      log.warn('readiness_check_failed', { dependency: dep, err: serializeError(err) });
    }
    await sleep(10_000);
  }
}

export function handleReadiness(): Response {
  if (isDraining()) return json({ status: 'draining' }, { status: 503 });

  const staleBefore = Date.now() - 30_000;
  const failing = CRITICAL_DEPS.filter((dep) => {
    const check = lastCheck.get(dep); // Map.get is T | undefined — no result yet means not ready
    return check === undefined || !check.ok || check.checkedAt < staleBefore;
  });

  // Cluster-internal endpoint: dep names in the body are for the operator debugging
  // a probe failure, not a public surface.
  if (failing.length > 0) return json({ status: 'not_ready', failing }, { status: 503 });
  return json({ status: 'ready' }, { status: 200 });
}
```

Three properties carry the design: probe frequency no longer multiplies dependency load (the loops run at *your* interval); every ping has its own deadline (`timeouts-everywhere` — a hung dependency becomes a failed check, not a hung probe); and a *stale* result counts as failing, so a wedged check loop can't report last week's green forever.

### Only can't-serve-without dependencies gate readiness

Readiness asks "can I serve my *core* path?", not "is everything I ever call up?". The database: yes, gate on it. The recommendations provider, the email service, the analytics sink: no — those failures degrade to fallbacks (`graceful-degradation-defaults`) while the instance keeps serving.

Gate readiness on a shared non-critical dependency and its outage makes *every* replica NotReady at the same moment — the load balancer evicts the entire fleet, and a degraded-but-working product becomes a total self-inflicted outage. The same logic applies to a `circuit-breaker-on-flaky-deps` being open on a garnish path: that's the degradation working, not unreadiness.

### Check your access, not their tree

A readiness check verifies *this instance's ability to use* a dependency — a pooled `SELECT 1`, a client-connection state — not the dependency's own full health, and never another service's deep-health endpoint. Recursive health checking is how one slow leaf makes an unrelated chain of services flap; each service owns exactly one hop.

### Draining is a readiness state

During shutdown the process is perfectly *live* (don't restart it — that aborts the drain) but must not receive new traffic. That's the `isDraining()` branch above, flipped by the SIGTERM handler in `graceful-shutdown-drain-on-sigterm`. This pairing is the reason the two endpoints can't be one: drain needs "not ready" and "alive" to be true simultaneously.

## Pressure Resistance

### "One /health endpoint is simpler"

Simpler until the orchestrator has to choose an action. Restart and stop-routing are opposite responses: a dependency outage wants stop-routing (or degradation) and *definitely not* restarts; a wedged process wants a restart. One endpoint forces one answer to both questions, and one of them will be wrong under fire.

### "Liveness should check the DB — a pod with no DB is useless"

It's not-ready, not dead. Readiness already removes it from rotation; restarting it additionally buys nothing, loses warm caches and in-flight work, and across the fleet turns a 40-second DB blip into a restart stampede. Liveness checks the process; dependencies are readiness's job.

### "Deep per-probe checks give the freshest signal"

Freshness that costs N×M×frequency synthetic load — the signal becomes the noise. A 10-second-old cached result is fresh enough for a routing decision, and the staleness bound in the handler keeps "cached" from silently becoming "ancient".

### "The stub 200 satisfies the platform, we'll deepen it later"

The stub converts the platform's safety mechanism into a rubber stamp: instances that can't serve stay in rotation with green dashboards. Later arrives via an incident whose headline is "traffic kept routing to a dead instance for 40 minutes".

### "Readiness should list every dependency, to be thorough"

Thoroughness inverts the availability math: each *non-critical* dependency added to readiness multiplies your fleet-wide eviction probability. Gate on the can't-serve-without set, degrade the rest.

## Red Flags

- One endpoint answering both probes.
- Any network call in a liveness handler.
- An unconditional 200 readiness handler.
- `await pingDb()` per probe request instead of a cached background check.
- A dependency ping without a timeout.
- Readiness listing garnish dependencies.
- No draining state, or liveness that fails during a graceful drain.
- A readiness handler calling another service's `/health`.

**All of these mean: the probes will answer the wrong question under failure — split the questions, cache the checks, gate on critical deps only.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The stub keeps the deploy green" | It keeps *broken instances* green too. That's the failure, automated. |
| "Restarting on DB failure can't hurt" | It can: fleet-wide restart storms, cold caches, aborted drains — and the DB is still down. |
| "Per-probe checks are more accurate" | They're a load generator with your fleet size as the multiplier. Cache with a staleness bound. |
| "Every dep in readiness is defensive" | It's offensive: any shared garnish outage now evicts the whole fleet. |
| "We'll notice a broken instance in the dashboards" | The probe exists so nobody has to notice. Make it tell the truth instead. |
| "Our framework's health plugin handles it" | Plugins give you the endpoint. Which deps gate it, and what liveness ignores, is still your design. |

## Related

- `graceful-shutdown-drain-on-sigterm` — the drain flips readiness while liveness stays green; the two skills are one lifecycle.
- `graceful-degradation-defaults` — a non-critical dependency failing means degrade, not NotReady; readiness gates only the core path.
- `circuit-breaker-on-flaky-deps` — an open breaker on a garnish dependency is degradation working, not a readiness failure.
- `timeouts-everywhere` — every dependency ping in the check loop carries its own deadline.

## Reference

- Kubernetes documentation, [*Configure Liveness, Readiness and Startup Probes*](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) — the probe semantics and the restart-vs-unroute distinction.
- Google SRE Book, ch. 20 ([*Load Balancing in the Datacenter*](https://sre.google/sre-book/load-balancing-datacenter/)) — health-based routing and the *lame duck* state: an instance that answers "not ready" while finishing existing work.
- Chris Richardson, [*Pattern: Health Check API*](https://microservices.io/patterns/observability/health-check-api.html) — the health endpoint as a first-class service contract.
