---
name: expand-contract-schema-migration
description: Use when writing a Drizzle migration that drops a column, renames a column, narrows a column type, or makes a nullable column NOT NULL. Use when changing a Postgres enum's values. Use when reshaping a JSON column's structure. Use when the diff between two deploys would have *old code* reading rows written by *new code* or vice versa. Use when a code reviewer says "we can do this in one PR" about a destructive schema change.
---

# Expand / Contract Schema Migration

## Overview

**A schema change that is not purely additive happens in at least two deploys.** First the **expand** deploy (add the new shape; write both; read either) lands and stabilizes. Only then does the **contract** deploy (stop writing the old shape; drop it) lands. Never combine expand and contract in one PR.

The "two deploys" framing assumes any modern deploy model where old and new instances overlap for seconds-to-minutes during rollout. Even if you don't believe in rollback, you can't avoid the overlap window — old code and new code run side-by-side, against the same database, while traffic shifts. That overlap is what the rule protects.

## The Iron Rule

```
NEVER ship a destructive schema change and the code that depends on it in the same deploy.
```

**No exceptions:**
- Not for "there's no traffic at 3 AM"
- Not for "atomic deploys handle this"
- Not for "it's a tiny table"
- Not for "we have backups"

## Why

A single-PR destructive migration looks like this:

```
deploy N    → app reads/writes `display_name`
              ALTER TABLE users DROP COLUMN display_name;
              ALTER TABLE users ADD COLUMN full_name text;
deploy N+1  → app reads/writes `full_name`
```

During the rollover window (seconds to minutes), function instances from deploy N are still serving requests. They expect `display_name`. The column is gone. Every request that hits an old instance hits `column "display_name" does not exist`. Logs light up. You roll back — but the column is gone *forever*. The rollback's old code can't read its own rows.

Even without rollback, every long-running request mid-deploy fails. Cron jobs scheduled before the migration ran and now executing post-migration fail. Any external service holding open connections fails.

The expand/contract sequence eliminates the window:

```
deploy N    → reads/writes display_name
deploy N+1  → ALTER TABLE users ADD COLUMN full_name text;
              reads display_name OR full_name (tolerant);
              writes BOTH display_name AND full_name;
              [STABILIZE — backfill old rows, monitor, wait]
deploy N+2  → reads/writes full_name only;
              ALTER TABLE users DROP COLUMN display_name;
```

Now any rollover window is safe: old code reads `display_name` (still there), new code reads `full_name` (already populated), both writes converge. Rollbacks work because no shape was destroyed without first replacing it.

This pattern is named "expand/contract" in *Refactoring Databases* (Ambler & Sadalage) and "parallel change" in Fowler's bliki. Both names describe the same discipline.

## Detection

You are violating the rule if any of these are true:

- A Drizzle migration in the same PR as the code that depends on it.
- A migration drops a column, narrows a type, adds `NOT NULL` to an existing column, or removes an enum value.
- The migration's "down" function would corrupt data (e.g., dropping a column whose data was never copied elsewhere).
- The PR description says "deploy migration first, then code" — informal sequencing instead of a structured expand/contract.
- A column rename done with a single `ALTER TABLE ... RENAME COLUMN`.
- A Postgres enum change that removes values (`ALTER TYPE ... DROP VALUE` doesn't exist; the only "removal" is a destructive rebuild).
- A migration's safety depends on "no traffic during deploy" — serverless platforms don't give you that.

## The Pattern

### Three categories — only the first is safe in one PR

| Change | Safe in one PR? | Pattern |
|---|---|---|
| **Pure additive**: new nullable column; new table; new index; new enum value (appended). | Yes | Land migration + code in one PR. |
| **Destructive**: drop column/table; narrow type; add `NOT NULL`; remove enum value; rename column. | **No** | Expand → stabilize → contract. |
| **Backfill required**: new `NOT NULL` column on a table with existing rows; type change that needs row-level transform. | **No** | Expand → backfill → flip → contract. |

### Anatomy of a rename — the canonical destructive change

```ts
// schema.ts — old state
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull(),
  display_name: text('display_name').notNull(), // ← to be renamed full_name
});
```

**Deploy E1 — Expand.** One PR.

```sql
-- migrations/0042_add_full_name.sql
ALTER TABLE users ADD COLUMN full_name text;
UPDATE users SET full_name = display_name WHERE full_name IS NULL; -- backfill
```

```ts
// schema.ts — additive only
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull(),
  display_name: text('display_name').notNull(),
  full_name: text('full_name'), // nullable for now; old rows backfilled
});

// Code: write to both; read full_name with display_name fallback.
async function updateUser(userId: UserId, name: string) {
  await db.update(users)
    .set({ display_name: name, full_name: name })
    .where(eq(users.id, userId));
}
function userName(u: typeof users.$inferSelect): string {
  return u.full_name ?? u.display_name;
}
```

[STABILIZE — wait at least one full deploy cycle + cron cycle. Verify monitoring shows `full_name` populated on 100% of writes. Backfill any stragglers.]

**Deploy E2 — Tighten.** One PR.

```sql
-- migrations/0043_full_name_required.sql
UPDATE users SET full_name = display_name WHERE full_name IS NULL; -- final sweep
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;
```

```ts
// schema.ts
full_name: text('full_name').notNull(), // now required

// Code: read full_name directly; keep dual-write for one more deploy.
function userName(u: typeof users.$inferSelect): string {
  return u.full_name;
}
```

**Deploy E3 — Contract.** One PR.

```sql
-- migrations/0044_drop_display_name.sql
ALTER TABLE users DROP COLUMN display_name;
```

```ts
// schema.ts
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull(),
  full_name: text('full_name').notNull(),
});

// Code: write to full_name only.
async function updateUser(userId: UserId, name: string) {
  await db.update(users).set({ full_name: name }).where(eq(users.id, userId));
}
```

Three PRs, three deploys, no rollover window where any traffic hits a missing column.

### NOT NULL on an existing column — backfill pattern

```sql
-- ❌ One-shot. Will fail if any row has NULL.
ALTER TABLE invoices ALTER COLUMN currency SET NOT NULL;

-- ✅ Two deploys.
-- E1: add default; backfill nulls
ALTER TABLE invoices ALTER COLUMN currency SET DEFAULT 'USD';
UPDATE invoices SET currency = 'USD' WHERE currency IS NULL;
-- [code continues to tolerate NULL via fallback]

-- E2 (next deploy): require it
ALTER TABLE invoices ALTER COLUMN currency SET NOT NULL;
-- [code now relies on non-null]
```

### Postgres enum — append safely, never remove

```sql
-- ✅ Safe — adding a value is additive
ALTER TYPE order_status ADD VALUE 'fulfilling' AFTER 'paid';

-- ❌ DROP VALUE doesn't exist in Postgres. The "removal" path:
--   1. Create new enum (without the value)
--   2. Add new column with the new enum type
--   3. Backfill, remapping the dead value to something safe
--   4. Swap columns
--   5. Drop old column and old enum
-- Four deploys minimum. Decide whether removing the value is actually worth it.
```

### Migration files are commits — they don't "roll back"

Drizzle generates numbered SQL files. Those files are *additive history* — never edit a shipped migration. A "rollback" of a destructive migration is a *new* migration that re-adds the column (empty) and a *new* backfill. Plan accordingly.

### Cron + schema changes — extra care

A cron mid-flight during a deploy will see old code reading the database mid-migration. If that cron writes data, it'd write the old shape. Two extras:

1. Pause the cron before the migration, resume after stabilization.
2. Make the cron's write path tolerant of both schemas during the expand window — same dual-write pattern as the route.

### The "monolithic migration in one PR" temptation

Migration tooling makes it easy to generate one file for a multi-step schema change. *Don't.* Split into separate PRs corresponding to expand / tighten / contract. Each PR is then reviewable and revertible on its own.

## Pressure Resistance

### "There's no traffic at 3 AM — we'll deploy then"

There is traffic. Cron jobs, scheduled tasks, retries from earlier failures, external partners' polling. Any deploy model with overlap assumes overlap — *every* deploy. "No traffic" isn't a real state.

### "Atomic deploys handle this"

Atomic deploys mean a *given request* sees a consistent function bundle. They do *not* mean all in-flight requests finish before the new bundle takes over — old instances keep serving requests for tens of seconds (and active connections can hold longer). The database is shared the entire time. Atomic ≠ instant.

### "One PR is faster"

It's faster to write and slower to debug. The first time you ship a one-shot destructive migration in production, the recovery time wipes out years of "saved" PR overhead. Three PRs is the cheap option.

### "Migration tooling generates the destructive SQL anyway"

It generates SQL that matches your schema diff. It's not deciding the migration *strategy* — you are. When the diff is destructive, you split the schema change into smaller diffs and run the tooling between them.

### "It's a tiny table, expand/contract is overkill"

Table size doesn't drive the rule — the *rollover window* does. A 12-row table still has zero rows for 3 seconds during a destructive migration if old code reads it during that window.

### "I'll back up the column to JSON before dropping"

That's expand/contract with a JSON column as the "new shape." Same rule; just messier execution. Use a real column.

## Red Flags

- A PR titled like "rename X → Y" with one migration file and code changes both.
- A migration that mixes `ADD COLUMN ... NOT NULL` and `DROP COLUMN` in one file.
- A code review comment "let's also bump the type to be stricter" — the bump and the strictening should be separate deploys.
- A migration file with no corresponding stabilization window plan in the PR description.
- The phrase "shouldn't be any traffic during deploy" in a PR description.
- An enum change that removes a value, in any form.

**All of these mean: the destructive change is collapsed into one deploy — split it before merging.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Atomic deploys handle this" | Atomic ≠ instant. Old instances drain for tens of seconds. |
| "It's not a real schema change" | If `DROP COLUMN`, `RENAME`, narrowing, or `NOT NULL`-tightening appears, it's a destructive change. |
| "We have backups" | Backups recover *data*; they don't unbreak *traffic during the rollover window*. |
| "One PR is cleaner" | One PR is cleaner *to write*. Three PRs are cleaner *to operate*. |
| "We can pause traffic" | On any modern serverless or multi-region stack, you can't reliably pause traffic. |
| "I'll do the migration manually after deploy" | Manual ad-hoc DDL is the version of expand/contract with no review and no test — strictly worse. |

## Reference

- Pramod Sadalage & Scott Ambler, *Refactoring Databases* (2006) — names the "expand/contract" pattern and catalogs the destructive schema changes that need it.
- Martin Fowler, ["Parallel Change"](https://martinfowler.com/bliki/ParallelChange.html) (2014) — the broader rename for the same pattern, applied to any breaking interface change.
- Jez Humble & David Farley, *Continuous Delivery* (2010), ch. 12 — the operational discipline this pattern enables (deploys decoupled from releases).
- Postgres docs on [`ALTER TABLE`](https://www.postgresql.org/docs/current/sql-altertable.html) and [`ALTER TYPE`](https://www.postgresql.org/docs/current/sql-altertype.html) — note that `DROP VALUE` is unsupported and `SET NOT NULL` rewrites the whole table; both inform the patterns above.
