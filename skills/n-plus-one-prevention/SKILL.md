---
name: n-plus-one-prevention
description: Use when fetching related data in loops. Use when seeing multiple queries for one request. Use when a list endpoint is slow but a detail endpoint is fast. Use when query count grows with result size. Use when reviewing code with `await` inside `.map()` or `.forEach()`.
---

# N+1 Query Prevention

## Overview

**Never let query count grow once per returned item.** Fetch related data with joins or bounded batch queries for one paginated result set.

N+1 is the pattern where you fetch N items, then make N more queries to get related data — 1 + N queries total. It's the most common database performance failure in apps, and it's almost always invisible until traffic grows enough to surface it.

## The Iron Rule

```
NEVER make query count proportional to result count. Join or batch each relation for a
bounded page, and chunk ID lists only to respect documented database parameter limits.
```

**These are not exceptions:**
- Not for "it's only a few items"
- Not for "the query is fast"
- Not for "we'll cache it"
- Not for "the ORM handles it"
- Not for "it's simpler to loop"
- Not for "it's an internal endpoint"

## Why

A list endpoint with N+1 doesn't fail — it gets *slower* as data grows. For 10 orders, you make 11 queries (1ms each = 11ms). For 1,000 orders, you make 1,001 queries (1ms each = 1 second). For 10,000 orders, the endpoint times out.

The bug ships silently and surfaces in production when:
- The user with the most data hits an unbearable latency.
- A monitoring alert fires on p99 for an endpoint that worked fine at p50.
- Connection-pool exhaustion: the DB rejects new queries while N+1 endpoints hold connections open.
- The DB's CPU saturates because every page load hits it 100×.

The fix is structural: **fetch all the data you need in one query** (via JOIN, `include`, relation-loading, or a typed batch fetch; for GraphQL resolvers, per-request batching — the dataloader pattern — is the standard fix). The query count becomes O(1) instead of O(N).

## Detection

You are violating the rule if any of these are true:

- `await` appears inside `.map()`, `.forEach()`, or a `for` loop body.
- A list endpoint takes 10× longer than the detail endpoint for one item.
- The DB query log shows the same query shape repeated N times per request.
- "Loading..." takes seconds on a list of items.
- A monitoring dashboard shows query count tracking row count.
- An ORM is configured with lazy loading by default and no eager-loading pattern is in use.

## The Pattern

### The canonical N+1 — related entity in a loop

```ts
// ❌ N+1: one query for orders, then N queries for customers.
const orderRows = await db.select().from(ordersTable);

const ordersWithCustomers = await Promise.all(
  orderRows.map(async (order) => {
    const customer = await db.select().from(customersTable)
      .where(eq(customersTable.id, order.customerId));
    return { ...order, customerName: customer[0]!.name };
  })
);
// Total queries: 1 + N. For 1,000 orders → 1,001 queries.
```

```ts
// ✅ One query with a JOIN.
const ordersWithCustomers = await db.select({
  order: ordersTable,
  customerName: customersTable.name,
})
  .from(ordersTable)
  .innerJoin(customersTable, eq(customersTable.id, ordersTable.customerId));
// Total queries: 1. Constant regardless of row count.
```

Most query builders express this naturally — Drizzle's `.innerJoin`, query builders' `include` clauses, raw SQL `JOIN`. Whatever the syntax, the rule is the same: one query for the data shape you need.

### Aggregates — count related rows in one query

```ts
// ❌ N+1: one query for users, then N queries for their order counts.
const userRows = await db.select().from(usersTable);
const usersWithCounts = await Promise.all(
  userRows.map(async (user) => {
    const count = await db.select({ count: sql<number>`count(*)` })
      .from(ordersTable)
      .where(eq(ordersTable.userId, user.id));
    return { ...user, orderCount: count[0]!.count };
  })
);
```

```ts
// ✅ One query with GROUP BY.
const usersWithCounts = await db.select({
  user: usersTable,
  // count(orders.id), NOT count(*): the left join emits one all-null row for a
  // user with zero orders, and count(*) would tally that row as 1. count() on
  // the joined column skips nulls, so zero-order users correctly read 0.
  orderCount: sql<number>`count(${ordersTable.id})`,
})
  .from(usersTable)
  .leftJoin(ordersTable, eq(ordersTable.userId, usersTable.id))
  .groupBy(usersTable.id);
```

### Multiple relations — fetch them in bounded batches

```ts
// ❌ N+1 squared: one query per order × two relations per order.
const orderRows = await db.select().from(ordersTable);
for (const order of orderRows) {
  const customer = await db.select().from(customersTable).where(eq(customersTable.id, order.customerId));
  const items = await db.select().from(orderItemsTable).where(eq(orderItemsTable.orderId, order.id));
  // ...
}
```

```ts
// ✅ Three queries total: orders, customers in one batch, items in one batch.
const orderList = await db.select().from(ordersTable).limit(PAGE_SIZE); // cursor omitted
const customerIds = orderList.map((o) => o.customerId);
const orderIds = orderList.map((o) => o.id);

const [customerList, itemList] = orderList.length === 0
  ? [[], []]
  : await Promise.all([
      db.select().from(customersTable).where(inArray(customersTable.id, customerIds)),
      db.select().from(orderItemsTable).where(inArray(orderItemsTable.orderId, orderIds)),
    ]);

// Then stitch in memory — O(N) work, but bounded by query count.
```

The pattern is **page, fetch each relation by ID list, stitch in memory**. This example uses three fixed round trips for one bounded page. It does not claim three queries have the same latency or data cost as one join—only that query count no longer grows once per item. If a page can exceed the database's bind-parameter limit, reduce the page size or chunk against a documented maximum; never send an unbounded `IN (...)` list.

A bonus: the batch rewrite also fixes an ordering bug the loop version often hides. `Promise.all(items.map(async ...))` that `push`es results resolves in *completion* order, not input order — so the output is non-deterministically shuffled. Stitching in memory by mapping over the *original* list (looking each relation up by ID) preserves order for free.

### When the per-item work is a function, not a query

The N+1 often hides behind a reusable enrichment/repository call, where a JOIN isn't reachable:

```ts
// ❌ enrichUser queries internally — N calls, one per user.
const enriched = await Promise.all(users.map((u) => enrichUser(u)));

// ✅ Write a batch version: one query for the whole set, stitch by id.
async function enrichUsers(users: User[]): Promise<EnrichedUser[]> {
  if (users.length === 0) return []; // inArray with [] throws — and there's nothing to fetch
  const profiles = await db.select().from(profilesTable)
    .where(inArray(profilesTable.userId, users.map((u) => u.id)));
  const byId = new Map(profiles.map((p) => [p.userId, p]));
  return users.map((u) => ({ ...u, profile: byId.get(u.id) ?? null }));
}
```

The durable fix is a batch-enrichment helper alongside the single-item one — `enrichUsers(users)` next to `enrichUser(user)` — so callers in a loop have a non-N+1 option.

### Watch the query count

```ts
// Useful telemetry: log query count per request, alert on outliers.
async function handleRequest(req: Request) {
  const { result, queryCount } = await withQueryCounter(() => handle(req));

  if (queryCount > 10) {
    log.warn('high_query_count', { path: req.url, queries: queryCount });
  }

  return result;
}
```

A list endpoint making >10 queries is suspicious. The exact threshold depends on the route, but the count must be request-scoped; a module-level counter races across concurrent requests.

## Pressure Resistance

### "It's only a few items"

10 today, 100 tomorrow, 10,000 next year. The N+1 doesn't grow — it *is*. The endpoint that works for the dev with 5 orders breaks for the customer with 5,000. Fix it while the data is small; you'll never notice the broken version was broken.

### "The query is fast"

1ms × 1,000 queries = 1 second. Add network round-trip overhead and connection-pool contention and you're at 3-5 seconds. One 5ms query with a JOIN beats 1,000 × 1ms queries every time.

"But I wrapped the loop in `Promise.all`, so they run concurrently and the latency doesn't add up." True — and that's the *more dangerous* shape, because measuring it looks fine. You've now fired N queries at once against a fixed-size connection pool: the real cost is pool exhaustion (other requests block waiting for a connection) and DB CPU saturation, not wall-clock latency for *this* request. A single batched query is still strictly better — it uses one connection, not N. Parallel N+1 trades visible latency for invisible pool/CPU pressure that surfaces as *other* endpoints slowing down.

### "We'll cache it"

The cache doesn't fix the underlying query. The first request is still slow. Cache misses are still slow. And the cache adds invalidation complexity. Fix the query first; cache after, if profiling shows it helps.

### "It's simpler to loop"

The loop is simpler to *write*. The query is simpler to *execute*. The user pays for execution simplicity, not write-time simplicity.

### "The ORM handles it"

Classic ORMs (Hibernate, ActiveRecord, Sequelize) default to lazy loading — they make N+1 *easier* to write, not harder. Modern TS clients (Prisma, Drizzle) have no lazy loading at all; there the N+1 is the explicit `await`-in-loop you wrote yourself. Either way the fix is the same: use the relation-fetching API (`include`, `with`, `joinRelated`, or whatever your tooling calls it) explicitly. Don't trust defaults — or loops.

### "It's an internal endpoint, doesn't matter"

Internal endpoints get hit by cron, by background jobs, by data exports, by retries. The N+1 surfaces when one of those processes runs over a large dataset.

## Red Flags

- `await` inside `.map()`, `.forEach()`, or a `for` loop body.
- A list endpoint's p99 latency tracks row count.
- DB query log shows the same query repeating with different parameters.
- A "Loading..." spinner on a list page that resolves in seconds.
- A `Promise.all` over a `.map` whose body queries the DB.
- A backend dev says "the query is slow when there are lots of items."

**All of these mean: N+1 is shipping — refactor to batch or JOIN.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's only a few items" | Data grows. Fix it while the cost is zero. |
| "The query is fast" | N×fast > 1×medium. The round-trip dominates. |
| "We'll cache it" | Cache doesn't fix bad queries; it postpones them. |
| "Looping is simpler" | Simpler to type, 100× slower to run. |
| "The ORM defaults are fine" | Lazy-loading ORMs make N+1 the default; Prisma/Drizzle make you write it yourself. Batch or eager-load explicitly. |
| "Premature optimization" | N+1 is a scaling defect — it breaks the query-count bound and only worsens as data grows. |

## Related

- `transaction-isolation` — both DB-query disciplines invisible until scale
- `race-conditions` — invisible-until-scale data-access bugs
- `cap-async-fan-out` — after collapsing per-item queries into one batch, cap the concurrency of what legitimately remains a fan-out
- `paginate-unbounded-reads` — the sibling bound: this skill caps query *count* per request; that one caps result *size* per query

## Reference

- Martin Fowler, *Patterns of Enterprise Application Architecture* (2002) — names the pattern and the canonical eager-loading vs. lazy-loading distinction; the "N+1 selects" problem traces to its lazy-load discussion.
- [PostgreSQL EXPLAIN ANALYZE](https://www.postgresql.org/docs/current/sql-explain.html) — use it to inspect the final query plan after batching or joining; use request-level query logs to prove query count dropped from N+1 to O(1).
