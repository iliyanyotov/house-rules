---
name: no-floating-promises
description: Use when a promise-returning call sits as a bare statement — not `await`ed, not returned, no `.catch()`. Use when a fire-and-forget side effect (analytics, email, cache warm) is launched with no rejection handler. Use when `void somePromise()` appears. Use when a promise is started early and awaited later, with other `await`s in between. Use when a process dies with an `unhandledRejection`, or an async failure leaves no trace in the logs.
---

# No floating promises

## Overview

**Every promise is observed: `await`ed, returned to a caller who will, or given a real `.catch` — including deliberate fire-and-forget.** A rejection nobody is listening for either vanishes silently or kills the process; both are worse than any handled failure.

The point is observation, not serialization. You don't have to `await` everything immediately — you have to guarantee that *someone* will see every rejection.

## The iron rule

```
NEVER leave a promise's rejection unobserved.
Await it, return it, or attach a .catch that reports — no bare statements.
```

**No exceptions:**
- Not for "it's just analytics / just a log write"
- Not for "`void` marks it as intentional"
- Not for "the global handler will catch it"
- Not for "it never fails in practice"

## Why

Since Node 15, an unhandled rejection is **fatal by default** (`--unhandled-rejections=throw`): a promise that rejects with no handler attached within a turn of the event loop raises as an uncaught exception and takes the process down. Before that — or with the mode downgraded — the failure is worse in a quieter way: it evaporates. No stack in your request logs, no error response, no retry. The write you thought happened, didn't.

```ts
// ❌ Fire-and-forget with no handler. If recordMetric rejects:
//    Node ≥15 default → process crashes, mid-request, for a metric.
//    warn mode / old Node → the rejection vanishes; nobody ever knows.
export async function completeOrder(order: Order): Promise<void> {
  recordMetric('order.completed', order.total); // bare statement — floating
  await db.update(orders).set({ status: 'completed' }).where(eq(orders.id, order.id));
}
```

```ts
// ✅ Deliberate fire-and-forget: launched without await, but OBSERVED.
//    The .catch reports through the boundary discipline — never an empty catch.
export async function completeOrder(order: Order): Promise<void> {
  recordMetric('order.completed', order.total).catch((err) => {
    logger.warn({ event: 'metric_write_failed', err }); // report-and-drop: optional work
  });
  await db.update(orders).set({ status: 'completed' }).where(eq(orders.id, order.id));
}
```

The rule is lint-enforceable and lint-enforced: `@typescript-eslint/no-floating-promises` ships in the recommended type-checked config. Turn it on; treat this skill as the *why* behind the squiggle and the discipline for the cases the linter accepts but runtime doesn't.

## Detection

You are violating the rule if any of these are true:

- A call whose return type is `Promise<...>` stands as a bare expression statement.
- `void somePromise()` is used to silence the linter — `void` documents intent but attaches **no handler**; the rejection is exactly as unhandled at runtime as before.
- An empty `.catch(() => {})` "handles" the rejection by deleting it — that's a silent swallow, which `swallow-deliberately-at-the-boundary` forbids.
- A promise is started early (`const p = doWork()`) and awaited after other `await`s — if an earlier `await` throws, `p`'s rejection has no handler and fires as unhandled.
- Inside a `try`, a promise is `return`ed without `await` — not an unobserved rejection (the caller observes it), but a trap in the same family: the local `catch` silently never fires, and the async stack trace drops this frame. Use `return await` inside `try`.
- The process's `unhandledRejection` hook logs and continues — that keeps rejections *observed* but defeats this skill's discipline: the hook is a crash reporter, not a recovery path (see below).

## The pattern

### Await it, return it, or catch it — the three legal states

```ts
await sendReceipt(order);          // 1. awaited — caller's try/catch sees failure
return sendReceipt(order);         // 2. returned — the caller now owns observation
sendReceipt(order).catch(report);  // 3. detached but observed — reporting handler
```

Anything else is floating. One refinement of state 2: **inside a `try` block, `return await`, not `return`** — a bare `return promise` is still observed by the caller, but it escapes the enclosing `catch` and drops the current frame from the eventual stack trace.

```ts
async function chargeWithFallback(order: Order): Promise<Receipt> {
  try {
    return await chargeCard(order); // ✅ await: rejection routes through THIS catch
  } catch {
    return chargeBackup(order);     // outside try — no local catch to route through
  }
}
```

### The start-early trap: a held promise is a live grenade

Starting work early for concurrency is good — but between start and `await`, the promise is unobserved. If anything in the gap throws, the held promise's rejection is unhandled.

```ts
// ❌ If validateInventory throws, `emailP` rejects into the void later.
const emailP = renderEmail(order);          // started, no handler attached
await validateInventory(order);             // throws → emailP is now floating
await sendEmail(await emailP);

// ✅ Start together, await together — every element gets a handler immediately.
const [email] = await Promise.all([renderEmail(order), validateInventory(order)]);
await sendEmail(email);
```

(`Promise.all` attaches a handler to *every* element, so non-winning rejections are observed — its real cost is that on first rejection the remaining work keeps running unwatched-by-you, which is a cancellation problem, not an unhandled-rejection one. See `cap-async-fan-out` for choosing `all` vs `allSettled`.)

### The global handler is a crash reporter, not a strategy

Register exactly one `process.on('unhandledRejection')` whose only jobs are to log the reason and shut down gracefully. An unhandled rejection means unknown state — a handler that logs-and-continues converts a loud bug into a slow one.

```ts
process.on('unhandledRejection', (reason) => {
  logger.error({ event: 'unhandled_rejection', reason });
  process.exitCode = 1;
  server.close(() => process.exit(1)); // drain, then die; the supervisor restarts
});
```

Keep the platform default (`throw`); never downgrade to `warn` or `none` to make the symptom stop.

## Pressure resistance

### "It's just analytics — who cares if it fails"

Then say so in code: `.catch` with a report-and-drop, per `swallow-deliberately-at-the-boundary`. "Don't care" is a decision that must be visible and observed — a floating promise is indistinguishable from a forgotten one, and on modern Node it can kill the process *because* you didn't care.

### "`void` marks it as intentionally not awaited"

`void` satisfies the linter and communicates intent to readers — and attaches nothing at runtime. The rejection is exactly as unhandled as before. Intent without a handler is a comment, not a fix: write `.catch(report)`.

### "The global unhandledRejection handler catches it"

By design, that handler's job is to crash you gracefully. Routing expected failures through a process-death path is not error handling. The safety net exists for the bug you missed, not the one you're writing on purpose.

### "Awaiting everything makes the code sequential and slow"

Observation ≠ serialization. Start work concurrently, then observe it together (`Promise.all` / `allSettled`) or hand it a reporting `.catch`. The rule constrains who *sees* the rejection, not when you await.

## Red flags

- A bare `somethingAsync();` statement — the signature returns `Promise`, nobody reads it.
- `void` prefixing a promise as the "fix" for a lint error.
- `.catch(() => {})` — observed and deleted is still deleted.
- `return promise` inside a `try`.
- A held promise crossing another `await` with no handler attached.
- `unhandledRejection` handler that doesn't exit.
- The lint rule `no-floating-promises` disabled file-wide.

**All of these mean: decide who observes this rejection, and wire it — await, return, or a reporting catch.**

## Common rationalizations

| Excuse | Reality |
|---|---|
| "It never rejects" | Every network/db/fs promise rejects eventually. That's when you need the handler most. |
| "`void` is the documented lint escape" | It documents. It doesn't handle. Lint-quiet ≠ runtime-safe. |
| "We'd rather not crash on a stray rejection" | Downgrading the mode hides bugs in unknown state. Fix the float, keep the crash. |
| "The catch would just log anyway" | Then it's a one-line `.catch(report)` — cheaper than the incident where nothing logged. |
| "Wrapping every call is noise" | Only *detached* work needs `.catch`. Awaited and returned promises are already observed. |

## Related

- `swallow-deliberately-at-the-boundary` — governs what the fire-and-forget `.catch` may do: report-and-drop for optional work, defer-to-retry for owed work; never an empty catch.
- `cap-async-fan-out` — the sibling rule for *how many* promises you start; this one is about *who observes* each of them.
- `errors-as-values` — once observed, route the failure through the typed error channel.
- `fail-fast` — the crash-on-unhandled default is fail-fast at process level; keep it.
- `race-conditions` — every `await` is a yield point; the start-early trap above is one more reason held promises need care.

## Reference

- [`@typescript-eslint/no-floating-promises`](https://typescript-eslint.io/rules/no-floating-promises/) — the lint rule (in the recommended type-checked config); its docs cover the exception for `void`. Companions: `no-misused-promises` and `return-await` (the `return await`-inside-`try` refinement above).
- [Node.js `process` events — `unhandledRejection` / `rejectionHandled`](https://nodejs.org/api/process.html#event-unhandledrejection) — the runtime semantics: no handler within one event-loop turn → unhandled.
- [Node.js v15.0.0 release notes](https://nodejs.org/en/blog/release/v15.0.0) — the default mode change from `warn` to `throw`: unhandled rejections are fatal by default on every Node since.
