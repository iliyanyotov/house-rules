---
name: steady-state-purge-unbounded-growth
description: Use when adding a table, cache, log index, file-storage bucket, queue, or session store that grows over time. Use when reviewing a cron that *creates* rows without any companion that *deletes* them. Use when a database table is described as "we never delete from it." Use when a Redis key family has no `EXPIRE`. Use when an idempotency-keys table has lived for >30 days and nobody can answer "how big is it now?" Use when a logs index has no retention policy.
---

# Steady-State: Purge Unbounded Growth

## Overview

**Anything a long-running system *creates* must have a paired *purge*.** A row gets a TTL or a scheduled cleanup. A cache key gets an `EXPIRE`. A log index gets a retention policy. A blob bucket gets a lifecycle rule. No exceptions, even for "small" data — *especially* for "small" data, because small things accumulate silently for years before they break in a way the team didn't plan for.

The system reaches *steady state* when growth equals purge — every day, the same number of rows enter and leave, and the table size oscillates around a fixed number. Until then, you have an unbounded growth problem regardless of the table's current size.

## The Iron Rule

```
NEVER ship a create-path without a purge-path. Every growing thing has a paired retention rule.
```

**No exceptions:**
- Not for "the table is tiny"
- Not for "we need the data for audit"
- Not for "Postgres handles big tables fine"
- Not for "we have backups"

## Why

Every production system reaches its limit through *unbounded growth*. Sometimes the limit is a hard one (storage cap, Postgres autovacuum stalling on a multi-billion-row table, Redis memory limit), sometimes a soft one (queries on a 50M-row table get slow enough that a feature degrades, then breaks), but every system has one.

The growth happens in the obvious places — user-data tables — and in the *non-obvious* places that bite worst:

- **Idempotency-key tables** that store one row per write request. After two years of traffic, billions of rows; queries get slow even though the table's "live" data is small.
- **Webhook-event logs** ingested for debugging. Useful for a week; toxic after a year.
- **Session rows** for users who haven't logged in since 2023.
- **Job-queue completed rows** kept "for audit." After 18 months, the completed queue is bigger than the live queue.
- **Email-send histories** kept "in case of complaints." Hundreds of millions of rows after a few years.
- **Soft-deleted rows** that the application correctly ignores but the database still scans on every query.
- **Cache entries** never invalidated because the data "rarely changes" — except when it does.
- **Error-tracker breadcrumb attachments** retained forever, eating the project's storage quota.
- **CI run artifacts** uploaded to object storage, never expired.

Each of these is a separate failure mode. The rule prevents all of them at once: **you cannot ship a write path without also shipping the purge path.** The purge can be a TTL, a retention policy, a scheduled cron, or a `pg_cron` job — but it has to exist.

## Detection

You are violating the rule if any of these are true:

- A new Drizzle migration adds a table with no `createdAt` column, no TTL, and no companion purge cron.
- A Redis key family is written without an `EX` / `PX` / `EXPIRE`.
- An idempotency-key table is mentioned in a PR but no retention policy is.
- A logs/events table description says "we keep this forever."
- A monitoring query "how big is table X?" returns a number that's grown every month for the past 6 months without a planned cap.
- A soft-delete pattern (`deletedAt IS NOT NULL` filtering) is in place without a hard-delete cleanup.
- A blob bucket has no lifecycle policy.
- A queue stores completed jobs indefinitely "for audit."
- A `pg_cron` extension is enabled but no scheduled cleanup job exists.

## The Pattern

### Every create-path has a purge-path — in the same PR

```ts
// ❌ Idempotency keys with no cleanup.
export const idempotencyKeys = pgTable('idempotency_keys', {
  key: text('key').primaryKey(),
  responseBody: jsonb('response_body').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

// Migration creates the table. Two years later: 800M rows.

// ✅ Same PR ships the retention cron.
export const idempotencyKeys = pgTable('idempotency_keys', {
  key: text('key').primaryKey(),
  responseBody: jsonb('response_body').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
}, (t) => ({
  createdAtIdx: index('idempotency_keys_created_at_idx').on(t.createdAt),
}));

// Daily cron: delete rows older than 7 days.
export async function handlePurgeIdempotencyKeys(req: Request) {
  if (req.headers.get('authorization') !== `Bearer ${env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 });
  }
  const cutoff = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  await db.delete(idempotencyKeys).where(lt(idempotencyKeys.createdAt, cutoff));
  return json({ ok: true });
}
```

The migration + the cron ship together. The retention window (7 days here) is named in the PR description and matches the *real* retry window.

### Cache entries — `EXPIRE` is required, not optional

```ts
// ❌ Redis cache with no TTL.
await redis.set(`user:${userId}`, JSON.stringify(user));

// ✅ Every cache write has a TTL.
await redis.set(`user:${userId}`, JSON.stringify(user), { ex: 60 * 60 }); // 1 hour
```

A cache entry without `EX` is a memory leak. Even if the data is "small," the keyspace can grow unbounded with old user IDs, partial migrations, and edge-case keys nobody remembered to clean.

### Soft-delete is not retention

```ts
// ❌ "We never delete; we mark deletedAt"
// ... 5 years later, 90% of every query's scan time is on soft-deleted rows.

// ✅ Soft-delete *with* a hard-delete cron.
// Mark on user action; permanently remove after the retention window.
async function purgeOldSoftDeletes() {
  const cutoff = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000); // 30 days
  await db.delete(orders).where(
    and(isNotNull(orders.deletedAt), lt(orders.deletedAt, cutoff))
  );
}
```

The 30-day window is the "user changed their mind" recovery window. After that, the soft delete graduates to a hard delete. This is the *only* way soft-delete doesn't become unbounded growth in disguise.

### Logs and events — retention at the index level

Logs go to a system with built-in retention (datasets configured per project, error-tracker event retention, log-aggregator policies). Postgres-based event tables get a partitioning + drop-old-partition pattern:

```sql
-- Partition by month; cron to DROP TABLE for partitions older than 90 days.
CREATE TABLE events (
  id uuid PRIMARY KEY,
  occurred_at timestamptz NOT NULL,
  payload jsonb NOT NULL
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2026_01 PARTITION OF events
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... one partition per month
```

Partitioning + dropping old partitions is *vastly* faster than `DELETE WHERE occurredAt < ...` on a huge table — the drop is a metadata operation; the delete rewrites every page.

### Blob storage — lifecycle policy

Configure a lifecycle rule on the bucket (object-expiration after N days). When the platform doesn't support per-bucket rules, run a cleanup cron:

```ts
async function purgeOldUploads() {
  const items = await storage.list({ prefix: 'tmp/' });
  const cutoff = Date.now() - 24 * 60 * 60 * 1000;
  for (const item of items) {
    if (new Date(item.uploadedAt).getTime() < cutoff) {
      await storage.delete(item.url);
    }
  }
}
```

Per-object retention beats "we'll clean it up eventually." `eventually` is "never" in practice.

### The purge cron is itself idempotent and bounded

```ts
// ✅ Purge in batches with a row limit.
async function purgeBatch() {
  const cutoff = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  const deleted = await db.execute(sql`
    DELETE FROM idempotency_keys
    WHERE key IN (
      SELECT key FROM idempotency_keys
      WHERE created_at < ${cutoff}
      LIMIT 10000
    )
  `);
  return { deleted: deleted.rowCount ?? 0 };
}
```

A purge that tries to delete 50M rows in one transaction *will* lock the table, time out, or both. Batch with `LIMIT`; run on a schedule that catches up. The cron is itself a steady-state pattern — each run deletes ≤N rows, so the cron never grows in cost as the table grows.

### Capacity math at PR time

A useful PR-checklist item:

> Storage growth: this table grows at ~X rows/day. With Y bytes/row, that's Z MB/year. The retention policy of W days caps it at V MB. Acceptable.

If you can't fill in those variables, the retention policy doesn't exist — the data has no plan.

## Pressure Resistance

### "The table is tiny — we'll deal with it later"

Today it's tiny. Tomorrow it's 50M rows. The cost of writing the cron *today* is one PR; the cost of retrofitting *in production* is an outage. Steady-state is cheaper in the small.

### "We need the data for audit / compliance"

Fine — but *the system that needs it for audit is not the same system serving live traffic*. Move audit data to a cold-storage system with its own retention rules. The hot operational store reaches steady state; the audit store has its own (much longer, regulated) retention.

### "Postgres handles big tables fine"

Postgres handles big tables fine for *workloads designed for them*. An OLTP table with point lookups by primary key scales well. A table that grows to 100M rows and then someone adds a `WHERE createdAt > ...` query without the index — that's where it breaks. Steady state limits the surface area for that class of bug.

### "The platform will warn us before limits"

It will — by either (a) failing your writes, or (b) sending an unexpected bill. Neither is a system you want to discover at the limit. Configure retention before you reach it.

### "We have backups; we can recover from anything"

Backups protect against *data loss*. They don't protect against *bad performance from too much data*. A 90-day-old idempotency-key table that's gotten too big doesn't get smaller from restoring a backup.

### "Soft delete is enough — `deletedAt IS NULL` everywhere"

Then your queries scan an ever-growing set of soft-deleted rows. The optimizer skips them, but the storage and the maintenance (autovacuum, index bloat) don't. Soft delete is a UX feature; it's not a retention strategy.

## Red Flags

- A new table migration with no companion purge cron in the same PR.
- A new Redis key family with no `EX`.
- The phrase "we never delete from this table" in a code review or design doc.
- A monitoring dashboard with a row-count line that goes up and to the right indefinitely.
- An idempotency-keys / sessions / events / webhook-history table older than 30 days with no retention policy.
- A cron named "purge-X" that runs once a year because nobody set up the schedule.
- A `DELETE FROM huge_table WHERE created_at < ...` without `LIMIT` — will time out.
- A `pg_cron` extension installed but nothing scheduled.
- An object-storage bucket with no lifecycle rule.

**All of these mean: the data has no plan — it will grow until it breaks something.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's small now" | Small grows. Steady state is independent of starting size. |
| "We'll add the cron when we need it" | "Need it" = customer-visible degradation. Too late. |
| "The database will handle it" | Until it doesn't. The cliff is silent until you fall off. |
| "Audit / compliance requires keeping it" | Then move it to cold storage with its own retention rules. The hot store doesn't need audit data. |
| "We have backups" | Backups protect against loss, not against accumulation. |
| "Cleanup is risky — what if we delete the wrong thing?" | Then the cron has a `WHERE` clause that you've tested. Risk lives in *manual* cleanups under outage pressure, not in scheduled cleanups under normal conditions. |
| "Soft-delete is enough" | Soft-delete + scheduled hard-delete is enough. Soft-delete alone is unbounded growth in disguise. |

## Related

- `idempotency-keys-on-writes` — the keys table grows; it needs a purge
- `dead-letter-and-replay` — the dead-letter table grows; it needs a purge

## Reference

- Michael Nygard, *Release It!* 2e (2018), ch. 5 — names "Steady State" as a stability pattern alongside Timeouts, Circuit Breaker, and Bulkheads. *"Every system should be able to run for a long time without operator intervention."*
- Postgres docs on [partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html), [`pg_cron`](https://github.com/citusdata/pg_cron), and autovacuum — the operational toolkit for actually achieving steady state on a Postgres table.
- Marc Brooker, ["Constant Work"](https://brooker.co.za/blog/2023/01/27/constant-work.html) — the broader case for designing systems whose cost per unit time is bounded regardless of accumulated state.
