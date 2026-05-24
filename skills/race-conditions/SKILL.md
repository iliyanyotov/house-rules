---
name: race-conditions
description: Use when writing code where multiple async operations touch the same state. Use when a `useEffect` performs a fetch. Use when a timer or interval mutates state. Use when a handler reads-then-writes a row. Use when calling `setState` inside a `setTimeout`, `setInterval`, or any callback that may fire after unmount. Use when "it works most of the time, but occasionally returns stale data" appears in a bug report.
---

# Race Conditions

## Overview

**When two operations can touch the same state at the same time, the order is unspecified** — design for both orderings or eliminate the concurrency.

For most JS/TS code, "concurrent" doesn't mean threads — it means *interleaving async continuations, late callbacks, and timer fires on a single event loop*. That's enough to produce real races.

## The Iron Rule

```
NEVER assume async operations land in the order you issued them. Cancel obsolete ones; prevent stale writes.
```

**No exceptions:**
- Not for "JavaScript is single-threaded"
- Not for "the user won't click that fast"
- Not for "I tested it, it works"
- Not for "the DB handles it"

## Why

JavaScript is single-threaded but **not race-free.** Every `await` is a suspension point where another callback can run. Every `setTimeout` callback can fire after its component has unmounted. Every handler can be invoked twice before the first finishes. Every `useEffect` cleanup can race with the effect it's cleaning up.

Races are pernicious because they almost always *work* during development. The bug surfaces in production with one user out of ten thousand, when the network is slow, or after a deploy when the function instance is warm. By then the developer has moved on and the original `console.log` has aged out.

The fix shape is consistent: **name the concurrent operations explicitly, cancel obsolete ones, and prevent stale writes.**

## Detection

You are violating the rule if any of these are true:

- A `useEffect` fetches data and calls `setState` without checking that the effect hasn't been superseded.
- A `setTimeout` / `setInterval` callback calls `setState` without a cleanup that cancels the timer.
- A handler does `select` → mutate-in-memory → `update` without a transaction or atomic operation.
- A button calls a mutation and doesn't disable itself; rapid double-click submits twice.
- A webhook handler can be invoked multiple times for the same event with no idempotency check.
- A bug report contains "this should never happen but sometimes does."

## The Pattern

### React: cancel obsolete fetches in `useEffect`

```tsx
// ❌ Stale-write race. If userId changes mid-flight, the older fetch can resolve
//    AFTER the newer one and overwrite fresh data with stale.
function ProfilePanel({ userId }: { userId: UserId }) {
  const [profile, setProfile] = useState<Profile | null>(null);
  useEffect(() => {
    fetchProfile(userId).then(setProfile);
  }, [userId]);
}

// ✅ Cancel the in-flight request when the effect re-runs or the component unmounts.
function ProfilePanel({ userId }: { userId: UserId }) {
  const [profile, setProfile] = useState<Profile | null>(null);
  useEffect(() => {
    const controller = new AbortController();
    fetchProfile(userId, { signal: controller.signal })
      .then(setProfile)
      .catch((err) => {
        if (err.name !== 'AbortError') throw err;
      });
    return () => controller.abort();
  }, [userId]);
}
```

The `controller.abort()` in the cleanup ensures a fetch from an older `userId` won't reach `setProfile` with stale data. The same `signal` pairs naturally with a timeout — one mechanism, two intents.

### React: timers that fire after unmount

```tsx
// ❌ If the timer fires after unmount, setState happens on an unmounted component.
function Countdown({ targetMs }: { targetMs: number }) {
  const [remaining, setRemaining] = useState(targetMs);
  useEffect(() => {
    const id = setInterval(() => setRemaining((r) => r - 1000), 1000);
    // ❌ no cleanup
  }, []);
}

// ✅ Clean up on unmount.
function Countdown({ targetMs }: { targetMs: number }) {
  const [remaining, setRemaining] = useState(targetMs);
  useEffect(() => {
    const id = setInterval(() => setRemaining((r) => r - 1000), 1000);
    return () => clearInterval(id);
  }, []);
}
```

If a callback can fire after unmount (long-running timer, subscription with no abort), guard with an `isMounted` ref:

```tsx
function ActiveCountdown({ expiryMs }: { expiryMs: number }) {
  const isMounted = useRef(true);
  useEffect(() => () => { isMounted.current = false; }, []);

  useExpiry(expiryMs, () => {
    if (!isMounted.current) return;
    // safe to setState
  });
}
```

### Server: read-then-write needs atomicity

```ts
// ❌ Two concurrent withdrawals both see balance=100, both subtract 50,
//    both write 50. Net: $50 deducted twice from a $100 starting balance.
async function withdraw(userId: UserId, amountCents: number) {
  const [user] = await users.findById(userId);
  if (user.balanceCents < amountCents) throw new InsufficientFundsError();
  await users.update(userId, { balanceCents: user.balanceCents - amountCents });
}

// ✅ Atomic SQL: server computes the new value; constraint prevents underflow.
async function withdraw(userId: UserId, amountCents: number) {
  const result = await db.execute(sql`
    UPDATE users
    SET balance_cents = balance_cents - ${amountCents}
    WHERE id = ${userId} AND balance_cents >= ${amountCents}
    RETURNING balance_cents
  `);
  if (result.length === 0) throw new InsufficientFundsError();
}

// ✅ Or: transaction with row-level lock when atomic SQL isn't enough.
async function withdraw(userId: UserId, amountCents: number) {
  await db.transaction(async (tx) => {
    const [user] = await tx
      .select().from(users).where(eq(users.id, userId))
      .for('update');
    if (user.balanceCents < amountCents) throw new InsufficientFundsError();
    await tx.update(users)
      .set({ balanceCents: user.balanceCents - amountCents })
      .where(eq(users.id, userId));
  });
}
```

### Server: double-submit and repeated webhooks

A button click can fire twice; a webhook delivery can repeat. Both are races where two writes arrive for the same logical event. The structural fix is *idempotency keys* — the server treats both arrivals as the same event and writes once. (See the `idempotency-keys-on-writes` skill.)

### Last-write-wins as a deliberate strategy

When you can't lock and can't make writes atomic, "last write wins" is sometimes fine — but only if you decide that **explicitly** and the data is naturally idempotent (a status update, a presence ping). Don't pick it by default; pick it when the domain allows it and document why.

## Pressure Resistance

### "JavaScript is single-threaded — there are no races"

Single-threaded means **no shared-memory races between threads**. It does *not* mean no concurrency. Every `await` is a yield point. Every callback is a future continuation that might interleave with another. The race surface is smaller than in multi-threaded languages, not zero.

### "The user won't click that fast"

Some will. A user with poor connection retries. A back button + refresh re-fires a request. An automated test or browser extension fires faster than any human. The race is determined by the *fastest* concurrent caller, not the typical one.

### "I'll add the cleanup later"

The cleanup is one line in the same `useEffect`. The "later" version requires remembering which effects need cleanups and revisiting them when something flakes. Cheaper now.

### "Transactions slow down writes"

Atomic SQL (`UPDATE ... WHERE balance >= ?`) is fast and race-free without explicit transactions. Transactions are for multi-row coordination. "Transactions are slow" is rarely about your actual workload — pick the right tool.

### "The user will see an error and retry"

If the race silently corrupts data, the user doesn't see an error — that's the whole problem. Visible errors are recoverable; silent corruption is the failure mode that hurts.

## Red Flags

- A `useEffect` with a `fetch(...)` and no `AbortController`.
- A `useEffect` returning nothing when it sets up a timer, subscription, or event listener.
- A `setState` inside a `setTimeout` / `setInterval` callback with no cleanup.
- A `select` → derived computation → `update` with no transaction or atomic SQL.
- A button without `disabled={isSubmitting}` that fires a mutation.
- A webhook handler with no dedup key.
- Bug reports mentioning "intermittent" or "happens occasionally" without a reproducer.
- "Works on my machine" but breaks under load.

**All of these mean: the race exists — name it and structurally eliminate it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "JS is single-threaded" | Concurrency ≠ parallelism. The event loop interleaves continuations. |
| "I tested it, it works" | You tested the happy interleaving. Races appear under timing pressure. |
| "It's a UI issue, not a backend one" | Both layers race in JS. UI races stale-overwrite state; backend races corrupt data. |
| "Optimistic locking is overkill" | For some operations, sure. For money / inventory / identity, never. |
| "We use Postgres, it handles this" | Postgres handles consistent writes *within* a transaction. Reads-then-writes split across two queries don't get the benefit unless wrapped. |
| "I'll add error handling for the corrupt state" | The corrupt state is the bug. Detection-and-recovery is more expensive than prevention. |

## Reference

- Dan Abramov, ["A Complete Guide to useEffect"](https://overreacted.io/a-complete-guide-to-useeffect/) — why effects + state are inherently racy and how cleanup eliminates the race.
- Marc Brooker (AWS), ["Caching with consistency"](https://brooker.co.za/blog/2020/11/16/cache.html) — broader systems take on read-then-write races.
- [Postgres docs on transaction isolation levels](https://www.postgresql.org/docs/current/transaction-iso.html) — required reading before claiming "the DB handles it."
