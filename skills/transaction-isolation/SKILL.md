---
name: transaction-isolation
description: Use when two database transactions can touch overlapping rows and the outcome depends on their order. Use when wrapping a read-then-write in `db.transaction(...)` and assuming that alone makes it safe. Use when a check across multiple rows (sum of bookings, count of seats, on-call coverage) must stay true after concurrent writes. Use when a balance, inventory count, or unique-slug assignment can be corrupted by two requests racing. Use when deciding between a row lock, an optimistic version column, and bumping the isolation level.
---

# Transaction Isolation

## Overview

**Wrapping work in a transaction does not, by itself, make it concurrency-safe. The isolation level decides which anomalies are possible — pick the level (or the lock) that prevents the specific anomaly your operation can hit, and name it.** Most databases default to `READ COMMITTED`, which still permits lost updates and write skew. A transaction at the wrong level corrupts data exactly as a non-transactional read-then-write does.

This skill is about the *database* concurrency layer: isolation levels, row locks, and optimistic version columns. App-level async races (a `useEffect` fetch, a timer after unmount, a double-clicked button) are a different execution model — see the `race-conditions` skill for those. The handoff: `race-conditions` covers making a **single-row** read-then-write atomic; this skill covers the failures that survive that — **multi-row invariants and isolation-level choice.**

## The Iron Rule

```
NEVER assume a transaction is safe at the default isolation level. Match the level — or a lock — to the anomaly the operation can hit.
```

**No exceptions:**
- Not for "it's wrapped in a transaction, so it's atomic"
- Not for "Postgres is ACID, it handles this"
- Not for "the window is too small to hit"
- Not for "we've never seen it corrupt in practice"

## Why

"ACID" promises that each transaction is *atomic, consistent, isolated, durable* — but **Isolation is a dial, not a guarantee.** SQL defines four levels (`READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`), and each *permits* a set of anomalies. The default on Postgres, MySQL/InnoDB, SQL Server, and Oracle is `READ COMMITTED` or weaker — which prevents dirty reads but still allows:

- **Lost update** — two transactions read a value, both modify it, the second overwrites the first. (Balance, counter, inventory.)
- **Write skew** — two transactions each read a set of rows, each makes a decision that's valid *given what it read*, and both commit — but together they violate an invariant neither could see the other breaking. (Two doctors both go off-call because each saw the other still on; two bookings for the last seat.)
- **Phantom read** — a transaction queries rows matching a predicate, another inserts a row matching it, and a re-query sees the new "phantom." (Re-checking a `COUNT(*)` you just validated.)

A transaction that does `read → decide → write` across rows is only safe if the level (or an explicit lock) forbids the anomaly it's exposed to. `BEGIN ... COMMIT` alone forbids none of the three at `READ COMMITTED`.

The fix is one of three tools, chosen by the anomaly — not "add a transaction and hope."

## Detection

You are violating the rule if any of these are true:

- A `db.transaction(...)` wraps a read-then-write and there's no row lock and no version check — the transaction is treated as sufficient on its own.
- An invariant spans **multiple rows** (a sum, a count, "at least one of X") and is validated with a plain `SELECT` before a write, at the default isolation level.
- A uniqueness rule ("one active subscription per user", "unique slug") is enforced by `SELECT ... if none, INSERT` rather than a DB constraint or a serializable transaction.
- Two concurrent requests on the same logical resource can both pass a check that should let only one through.
- A retry loop around a write does *not* handle the serialization-failure error code (`40001` on Postgres) — so a `SERIALIZABLE` transaction that aborts is treated as a hard failure.
- The phrase "it's in a transaction, so it's safe" appears without naming an isolation level or a lock.

## The Pattern

### A transaction at `READ COMMITTED` does not stop a lost update

```ts
// ❌ Wrapped in a transaction — and still loses updates. Both transactions read
//    balance=100 before either writes; the second commit overwrites the first.
await db.transaction(async (tx) => {
  const [acct] = await tx.select().from(accounts).where(eq(accounts.id, id));
  await tx.update(accounts)
    .set({ balanceCents: acct.balanceCents - amount })
    .where(eq(accounts.id, id));
});
```

The transaction gives atomicity and durability. It does **not** give isolation against a concurrent reader at the default level. For a single row, the fixes are the ones in `race-conditions` — atomic SQL (`SET balance = balance - ? WHERE balance >= ?`) or `SELECT ... FOR UPDATE`. The rest of this skill is about the cases those don't cover.

### Pessimistic: lock the rows you're about to decide on (`SELECT ... FOR UPDATE`)

```ts
// ✅ Take a row lock before reading the value you'll write back. The second
//    transaction blocks on FOR UPDATE until the first commits, then sees fresh data.
await db.transaction(async (tx) => {
  const [acct] = await tx
    .select().from(accounts).where(eq(accounts.id, id))
    .for('update');                       // SELECT ... FOR UPDATE
  if (acct.balanceCents < amount) throw new InsufficientFundsError();
  await tx.update(accounts)
    .set({ balanceCents: acct.balanceCents - amount })
    .where(eq(accounts.id, id));
});
```

Use when contention is **high** and you'd rather serialize the conflicting writers than retry them. The lock is held to the end of the transaction — keep the transaction short, and always lock rows **in a consistent order** across the codebase, or two transactions locking the same two rows in opposite orders will **deadlock**.

### Optimistic: a version column, retried on conflict

```ts
// ✅ No lock held across the think-time. Read the version, write only if it's
//    unchanged; if the row moved under you, the UPDATE matches 0 rows — retry.
async function applyDiscount(id: AccountId, pct: number): Promise<void> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const [acct] = await db.select().from(accounts).where(eq(accounts.id, id));
    const next = Math.round(acct.balanceCents * (1 - pct));

    const updated = await db.update(accounts)
      .set({ balanceCents: next, version: acct.version + 1 })
      .where(and(eq(accounts.id, id), eq(accounts.version, acct.version)))
      .returning({ id: accounts.id });

    if (updated.length > 0) return;        // our version won
    // someone else committed first — loop, re-read, recompute
  }
  throw new ConcurrencyConflictError();
}
```

Use when contention is **low** and you don't want to hold locks (long think-time, user-facing edit, optimistic UI). The cost is a retry on conflict — which means the operation must be **safe to recompute from scratch**.

### Multi-row invariant: write skew needs `SERIALIZABLE` (or a lock on the conflict set)

```ts
// ❌ Write skew. Each transaction checks "is at least one other doctor on call?"
//    Both read 2 doctors on call, both conclude it's safe to go off, both commit.
//    Result: zero coverage — an invariant neither transaction could see breaking.
async function goOffCall(doctorId: DoctorId, shiftId: ShiftId) {
  await db.transaction(async (tx) => {
    const onCall = await tx.select().from(shifts)
      .where(and(eq(shifts.shiftId, shiftId), eq(shifts.onCall, true)));
    if (onCall.length <= 1) throw new MinimumCoverageError();   // checks OTHER rows
    await tx.update(shifts).set({ onCall: false })
      .where(eq(shifts.doctorId, doctorId));
  });
}

// ✅ SERIALIZABLE makes the database detect the read/write conflict between the
//    two transactions and abort one with a serialization failure (40001).
async function goOffCall(doctorId: DoctorId, shiftId: ShiftId) {
  await withSerializableRetry(() =>
    db.transaction(async (tx) => {
      await tx.execute(sql`SET TRANSACTION ISOLATION LEVEL SERIALIZABLE`);
      const onCall = await tx.select().from(shifts)
        .where(and(eq(shifts.shiftId, shiftId), eq(shifts.onCall, true)));
      if (onCall.length <= 1) throw new MinimumCoverageError();
      await tx.update(shifts).set({ onCall: false })
        .where(eq(shifts.doctorId, doctorId));
    }),
  );
}
```

`FOR UPDATE` on the rows you read **does not** help here — the anomaly is between what one transaction *reads* and what the other *writes*, and the dangerous case is often rows that don't exist yet (phantoms). The two real options: lift the transaction to `SERIALIZABLE` (Postgres's SSI detects the conflict and aborts one), or take an explicit lock on the *whole conflict set* (e.g. a `FOR UPDATE` on the parent shift row, or an advisory lock keyed on `shiftId`).

### `SERIALIZABLE` is a contract: you MUST retry on serialization failure

```ts
// ✅ SERIALIZABLE transactions can abort with SQLSTATE 40001 (or 40P01 deadlock).
//    That's not a bug — it's the level doing its job. Retry the WHOLE transaction.
async function withSerializableRetry<T>(run: () => Promise<T>, max = 5): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try {
      return await run();
    } catch (e) {
      if (isSerializationFailure(e) && attempt < max) {
        await sleep(backoffWithJitter(attempt));   // see retry-with-jitter-and-budget
        continue;
      }
      throw e;
    }
  }
}
```

Choosing `SERIALIZABLE` without a retry loop trades a data-corruption bug for an intermittent-500 bug. The retry is part of the pattern, not an optional extra. (Back off with jitter — see `retry-with-jitter-and-budget`.)

### Uniqueness: prefer a constraint over check-then-insert

```ts
// ❌ Two requests both SELECT (no active sub), both INSERT — now two active subs.
const [existing] = await db.select().from(subs)
  .where(and(eq(subs.userId, userId), eq(subs.status, 'active')));
if (!existing) await db.insert(subs).values({ userId, status: 'active' });

// ✅ Let the database enforce it. A partial unique index makes the second
//    INSERT fail atomically — no read, no race, no isolation-level reasoning.
// migration: CREATE UNIQUE INDEX one_active_sub ON subs (user_id) WHERE status = 'active';
try {
  await db.insert(subs).values({ userId, status: 'active' });
} catch (e) {
  if (isUniqueViolation(e)) throw new AlreadySubscribedError();
  throw e;
}
```

A unique constraint is the cheapest concurrency control there is: the database enforces the invariant atomically regardless of isolation level. Reach for it before reaching for `SERIALIZABLE` whenever the invariant is expressible as a constraint.

## Choosing the tool

| Situation | Tool |
|---|---|
| Single-row counter / balance | Atomic `UPDATE ... WHERE` (see `race-conditions`) |
| Single resource, high contention, short critical section | `SELECT ... FOR UPDATE` |
| Single resource, low contention, long think-time / optimistic UI | Version column + retry |
| Invariant spanning multiple/absent rows (write skew, phantom) | `SERIALIZABLE` + retry, or lock the whole conflict set |
| Uniqueness ("one active X", "unique slug") | DB unique constraint (partial index if conditional) |

## Pressure Resistance

### "It's wrapped in a transaction, so it's atomic and safe"

Atomic and isolated are different properties. A transaction guarantees all-or-nothing *durability*; it does not, at the default level, guarantee that a concurrent transaction didn't read the same rows and write between your read and your write. Name the isolation level or the lock — "it's in a transaction" is not an answer.

### "Postgres is ACID — it handles concurrency"

It hands you the *tools* (isolation levels, row locks, SSI, constraints). It does not pick them for you. The default is `READ COMMITTED`, which permits lost updates and write skew. ACID is the menu, not the meal.

### "SERIALIZABLE is too slow / we can't afford it"

Then use a narrower tool — a row lock or a version column — for that specific operation. But "too slow" is usually unmeasured; `SERIALIZABLE` on Postgres (SSI) is optimistic and only pays at commit on actual conflict. Measure your contention before assuming it's too expensive, and scope it to the operations that need it.

### "The window is microseconds — it'll never happen"

Concurrency bugs are determined by traffic, not by window size. At 1,000 req/s the microsecond window is hit thousands of times a day. "Never seen it" means "haven't looked at the right row under load," not "can't happen."

### "Optimistic locking adds a retry loop — that's complexity"

The alternative is silent corruption, which is far more complexity — to detect, to reconcile, to apologize for. A bounded retry on a version conflict is a known, contained cost. (And it's the same retry discipline you already apply to flaky dependencies.)

### "We'll add a uniqueness check in the application"

The application check races with itself across processes. Two instances both pass it. The only check that can't race is the one the database makes atomically — a constraint. Application-level uniqueness is a constraint you forgot to write.

## Red Flags

- A `db.transaction(...)` around a read-then-write with no `FOR UPDATE`, no version column, and no elevated isolation level.
- A `SELECT COUNT(*)` / `SUM(...)` validated, then a write that depends on the count staying the same — at the default level.
- `SELECT ... if empty then INSERT` for a uniqueness rule, instead of a unique constraint.
- A `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE` with no surrounding retry on `40001`.
- Two code paths that lock the same two tables/rows in opposite orders (deadlock waiting to happen).
- "We use transactions everywhere" offered as the concurrency-safety story, with no mention of levels or locks.

**All of these mean: the operation can hit an anomaly the current isolation level permits — pick the level or lock that forbids it, and name it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "A transaction makes it atomic" | Atomicity ≠ isolation. The default level still permits lost updates and write skew. |
| "The DB is ACID" | ACID gives you the dial; you still have to turn it. The default is the weak end. |
| "FOR UPDATE fixes it" | For single rows you read, yes. For write skew across rows (or phantoms), no — that needs SERIALIZABLE or a conflict-set lock. |
| "SERIALIZABLE is too slow" | Measure first; scope it to the operation. Silent corruption is the more expensive option. |
| "We check uniqueness in code" | App checks race across processes. Only a DB constraint is atomic. |
| "It's never corrupted in prod" | You haven't queried for the corruption, or traffic hasn't peaked. Absence of evidence isn't isolation. |

## The Bottom Line

**A transaction without a chosen isolation level (or lock) is a guess, not a guarantee.**

Pick the tool by the anomaly: atomic `UPDATE` for one-row counters, `FOR UPDATE` for hot single resources, a version column for low-contention edits, `SERIALIZABLE` + retry for multi-row invariants, a unique constraint for uniqueness. Name the choice in the code, so the next reader knows which anomaly you defended against.

## Reference

- Martin Kleppmann, *Designing Data-Intensive Applications* (2017), ch. 7 ("Transactions") — the definitive treatment of isolation levels, the anomalies each permits, and write skew in particular. The source for this skill's framing.
- [PostgreSQL docs — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html) — the exact guarantees of each level on Postgres, and how Serializable Snapshot Isolation (SSI) detects conflicts.
- Peter Bailis et al., ["Highly Available Transactions: Virtues and Limitations"](http://www.bailis.org/papers/hat-vldb2014.pdf) (2014) — what isolation levels real databases actually provide versus what the standard says, and why the defaults are weak.
