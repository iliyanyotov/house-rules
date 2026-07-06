---
name: graceful-shutdown-drain-on-sigterm
description: Use when a service has no SIGTERM handler, or its handler is `process.exit(0)`. Use when every deploy or scale-down produces a burst of 502s, ECONNRESET, or half-finished jobs. Use when `server.close()` is the whole shutdown story and the process hangs until the platform kills it. Use when a queue consumer dies mid-message on every rollout. Use when logs show the platform's kill (OOM-style abrupt exit, grace period exceeded) as the normal end of every instance's life.
---

# Graceful Shutdown: Drain on SIGTERM

## Overview

**On SIGTERM, a service drains: flip readiness to unhealthy, stop accepting new work, finish in-flight work under a deadline, close connections, then exit cleanly.** SIGTERM arrives on every deploy, every scale-down, every node rotation — it is the *routine* way an instance dies, and dropping live requests on it turns every release into a micro-outage.

The platform's SIGKILL at the end of the grace period is the backstop, never the plan.

## The Default Rule

```
On SIGTERM: fail readiness → stop taking new work → drain in-flight work
under a deadline → close clients → exit 0. Treat SIGTERM as "finish what
you hold", not "die now".
```

This is a default, not an invariant — the named exceptions:

- **One-shot scripts and cron jobs** holding no in-flight requests: exiting promptly *is* the drain.
- **Runtimes that never deliver SIGTERM to your code** (some serverless platforms freeze or reap instances directly): rely on the platform's own request draining, and keep writes idempotent for the cases it misses.
- **A poisoned process** that cannot safely finish its work should exit fast rather than drain corrupt state — that judgment is what the drain deadline encodes.

## Why

Two failures, one per direction of the mistake:

- **Exit immediately** (`process.exit(0)` in the handler, or no handler under a process manager that then kills): every in-flight request dies as a connection reset, every in-progress job dies mid-write. At one deploy per day that's a daily, self-inflicted error burst — and users learn that "around lunchtime the app glitches".
- **Never exit** (no handler, or a drain that hangs): the process ignores SIGTERM until the grace period lapses and SIGKILL lands — which is the exit-immediately failure again, just 30 seconds later and with DB transactions and locks torn down mid-flight.

There's a subtlety that breaks naive handlers on orchestrated platforms: SIGTERM and load-balancer endpoint removal happen *in parallel*, not in order. New requests keep arriving for a few seconds after the signal. A handler that closes the listener instantly refuses those requests; the correct sequence fails readiness first, waits briefly for routing to catch up, and only then stops accepting.

## Detection

You are violating the rule if any of these are true:

- No `process.on('SIGTERM', ...)` anywhere in the service.
- The handler is `process.exit(0)` — an instant drop of everything in flight.
- The handler calls `server.close()` and nothing else — `close()` waits on active requests with no deadline (and on Node <19 it never released idle keep-alive sockets at all), and it flips no readiness, stops no workers, closes no pool.
- Readiness keeps reporting healthy while the process is shutting down.
- A queue consumer has no stop path — it dies holding an unacked or half-processed message on every deploy.
- Deploy timestamps line up with 502/ECONNRESET spikes in the dashboard.
- The DB pool is closed before in-flight requests finish using it.

## The Pattern

### The full sequence

```ts
import type { Server } from 'node:http';

const DRAIN_MS = {
  // Size to YOUR probe config: readiness periodSeconds × failureThreshold + controller
  // lag — 5s fits a 2s/2-failure probe; a 10s/3 probe needs ~35s and a bigger grace period.
  readiness_propagation: 5_000,
  in_flight_requests: 15_000,
  total: 25_000, // stay under the platform's grace period (Kubernetes default: 30s)
} as const;

let draining = false;
export function isDraining(): boolean {
  return draining; // the readiness endpoint returns 503 while this is true
}

export function registerShutdown(server: Server): void {
  const onSignal = () => {
    void shutdown(server).catch((err: unknown) => {
      log.error('shutdown_failed', { err: serializeError(err) });
      process.exit(1);
    });
  };
  process.on('SIGTERM', onSignal);
  process.on('SIGINT', onSignal); // local parity: Ctrl-C drains the same way
}

async function shutdown(server: Server): Promise<void> {
  if (draining) return;
  draining = true;
  log.info('shutdown_started', {});

  // Backstop: if the drain wedges, exit before the platform SIGKILLs blind.
  const deadline = setTimeout(() => {
    log.error('shutdown_deadline_exceeded', {});
    process.exit(1);
  }, DRAIN_MS.total);
  deadline.unref();

  // 1. Readiness now fails (isDraining() === true). Wait for routing to actually
  //    stop — on pod deletion Kubernetes removes the endpoint in parallel with the
  //    SIGTERM (the readiness flip mainly covers non-orchestrated LBs and scale-in),
  //    and either way new requests keep arriving for a few seconds after the signal.
  await sleep(DRAIN_MS.readiness_propagation);

  // 2. Stop accepting; drain in-flight HTTP. close() waits for active requests with
  //    no deadline (pre-Node-19 it also wedged on idle keep-alive sockets) — release
  //    idle explicitly and force any stragglers at the deadline.
  const closed = new Promise<void>((resolve) => {
    server.close(() => resolve());
  });
  server.closeIdleConnections();                    // Node ≥18.2
  const stragglers = setTimeout(() => server.closeAllConnections(), DRAIN_MS.in_flight_requests);
  await closed;
  clearTimeout(stragglers);

  // 3. Background work: stop pulling new jobs; finish (or release) the one in hand.
  await queueConsumer.stop({ finishCurrentJob: true });

  // 4. Only now close shared clients — in-flight work needed them until here.
  await dbPool.end();

  log.info('shutdown_complete', {});
  process.exit(0);
}
```

The ordering is the content: readiness first, listener second, workers third, clients last. Reversing any pair reintroduces a failure (closing the pool early starves live requests; closing the listener before routing updates refuses routed traffic).

### `server.close()` has no deadline

`server.close()` stops *new* connections and waits for existing requests to finish — with no time bound. On Node <19 it was worse: idle keep-alive sockets counted as "existing connections" that never end, so the close callback never fired and the process hung to SIGKILL; since Node 19, `close()` releases idle keep-alive sockets itself. Two gaps remain on any version: an active request that won't finish holds shutdown open indefinitely — `server.closeAllConnections()` at the drain deadline cuts it — and readiness/workers/pool are entirely outside `close()`'s job. The explicit `server.closeIdleConnections()` (Node ≥18.2) in the sequence above is defensive: a no-op on modern Node, the fix on 18.x.

### Budget the deadlines against the platform's grace period

The platform gives one total grace period (Kubernetes: `terminationGracePeriodSeconds`, default 30s). Your internal budgets must sum below it: propagation wait + request drain + worker stop + client close < grace period, with margin. A drain budgeted at exactly the grace period ends in SIGKILL on the slow day, which is the day it matters.

### Queue consumers: stop pulling, settle the message in hand

HTTP drains by refusing new connections; a consumer drains by not asking for the next message. Finish the current message if it fits the deadline; otherwise release/nack it so redelivery is prompt rather than waiting out a visibility timeout. Either way the message is redelivered at least once across deploys eventually — which is why the handler was already idempotent (`idempotency-keys-on-writes`).

### The deadline will kill something, someday

A drain deadline is a promise to eventually drop work. That's the correct trade — but it means anything the deadline can kill must be retryable: idempotent writes, redeliverable messages, resumable jobs. Graceful shutdown reduces dropped work from "every deploy" to "rare"; idempotency covers the rare.

## Pressure Resistance

### "The platform gives us 30 seconds anyway"

The grace period is a countdown to SIGKILL, not a drain. Nothing in those 30 seconds stops routing, refuses new work, or closes your pool unless your handler does it. An unhandled SIGTERM just means dying at second 30 instead of second 0 — with the same dropped requests.

### "Deploys are rare, a few errors are fine"

Deploys are the *most frequent* planned event in a healthy service's life — plus autoscaling scale-downs and node rotations you don't schedule. "A few errors per deploy" times daily deploys is a permanent background error rate with your release cadence as the fingerprint.

### "We just retry on the client, server drain is redundant"

Client retries paper over dropped *reads*. A dropped mutation mid-handler leaves half-applied state that a retry may double-apply — so retries make draining *and* idempotent handlers more necessary, not less. And half of your traffic (webhooks, partners) retries on schedules you don't control.

### "server.close() handles it"

It handles the accept side. It does not flip readiness, does not wait for routing, does not stop queue consumers, does not close the pool — and it waits on active requests with no deadline. It's one line of a six-line story.

### "Graceful shutdown code never runs in dev, it'll rot"

Wire SIGINT to the same path so every local Ctrl-C exercises it, and watch for `shutdown_complete` in deploy logs — its absence is the regression alarm.

## Red Flags

- No SIGTERM handler, or a handler containing `process.exit(0)`.
- `server.close()` with no `closeIdleConnections` / force-close deadline.
- Readiness that can't say "draining".
- A shutdown path with no overall deadline — it will wedge to SIGKILL eventually.
- The DB pool closed first, or never.
- 5xx/reset spikes that correlate with deploy timestamps.
- A queue consumer with no `stop()`.

**All of these mean: instances die dirty on every rollout — implement the drain sequence and budget it under the grace period.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The platform handles termination" | The platform delivers a signal and a countdown. The drain is entirely yours. |
| "It's only a second of errors" | Per instance, per deploy, per scale-down, forever — and visible to users each time. |
| "Our requests are fast, nothing is in flight" | Something is always in flight at your traffic's tail. That tail meets every deploy. |
| "server.close() is graceful shutdown" | It's the accept-side sixth of it, and it waits on active requests with no deadline. |
| "We'll add it when we adopt Kubernetes" | Any rolling deploy or process manager sends SIGTERM today. The bursts are already in your dashboards. |
| "Exit fast, the queue redelivers" | Redelivery saves the message, not the user waiting on the half-dead HTTP request beside it. |

## Related

- `health-and-readiness-endpoints` — the readiness endpoint this skill flips; liveness must *not* fail during a drain, or the orchestrator restarts a healthy shutdown.
- `idempotency-keys-on-writes` — whatever the drain deadline kills gets retried or redelivered; idempotency is what makes that safe.
- `timeouts-everywhere` — the drain deadline is the same discipline pointed inward: no unbounded wait, even on your own exit path.

## Reference

- Yoni Goldberg et al., [*Node.js Best Practices*](https://github.com/goldbergyoni/nodebestpractices) — the graceful-shutdown guidance in the Docker/production sections: handle SIGTERM, drain, close resources, exit intentionally.
- Kubernetes documentation, [*Pod Lifecycle — termination of Pods*](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) — SIGTERM, the grace period, and the parallel endpoint-removal race.
- Node.js documentation, [`http.Server`](https://nodejs.org/api/http.html) — `server.close()`, `closeIdleConnections()`, `closeAllConnections()` semantics the drain sequence depends on.
- Daniele Polencic, [*Graceful shutdown in Kubernetes*](https://learnkube.com/graceful-shutdown) (LearnKube) — the readiness-propagation race explained in detail: endpoint removal and the kubelet's SIGTERM are issued simultaneously.
