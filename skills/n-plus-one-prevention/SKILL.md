---
name: n-plus-one-prevention
description: Use when fetching related data in loops. Use when seeing multiple queries for one request. Use when a list endpoint is slow but a detail endpoint is fast. Use when query count grows with result size. Use when reviewing code with `await` inside `.map()` or `.forEach()`.
---

# N+1 Query Prevention

## Overview

**Never query in a loop. Fetch related data in a single query.**

N+1 is the pattern where you fetch N items, then make N more queries to get related data — 1 + N queries total. It's the most common database performance failure in apps, and it's almost always invisible until traffic grows enough to surface it.

## The Iron Rule

```
NEVER put a database query inside a loop. One query for the list; one query for each relation.
```

**No exceptions:**
- Not for "it's only a few items"
- Not for "the query is fast"
- Not for "we'll cache it"
- Not for "the ORM handles it"

## Why

A list endpoint with N+1 doesn't fail — it gets *slower* as data grows. For 10 orders, you make 11 queries (1ms each = 11ms). For 1,000 orders, you make 1,001 queries (1ms each = 1 second). For 10,000 orders, the endpoint times out.

The bug ships silently and surfaces in production when:
- The user with the most data hits an unbearable latency.
- A monitoring alert fires on p99 for an endpoint that worked fine at p50.
- Connection-pool exhaustion: the DB rejects new queries while N+1 endpoints hold connections open.
- The DB's CPU saturates because every page load hits it 100×.

The fix is structural: **fetch all the data you need in one query** (via JOIN, `include`, relation-loading, or a typed batch fetch). The query count becomes O(1) instead of O(N).

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
    return { ...order, customerName: customer[0].name };
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
    return { ...user, orderCount: count[0].count };
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

### Multiple relations — fetch them all in one shot

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
const orderList = await db.select().from(ordersTable);
const customerIds = orderList.map((o) => o.customerId);
const orderIds = orderList.map((o) => o.id);

const [customerList, itemList] = await Promise.all([
  db.select().from(customersTable).where(inArray(customersTable.id, customerIds)),
  db.select().from(orderItemsTable).where(inArray(orderItemsTable.orderId, orderIds)),
]);

// Then stitch in memory — O(N) work, but bounded by query count.
```

The pattern is **fetch by ID list, stitch in memory**. Three queries scale the same as one; one query per relation per item does not.

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

### "We'll cache it"

The cache doesn't fix the underlying query. The first request is still slow. Cache misses are still slow. And the cache adds invalidation complexity. Fix the query first; cache after, if profiling shows it helps.

### "It's simpler to loop"

The loop is simpler to *write*. The query is simpler to *execute*. The user pays for execution simplicity, not write-time simplicity.

### "The ORM handles it"

Most ORMs default to lazy loading — they make N+1 *easier* to write, not harder. The fix is to use the eager-loading API (`include`, `with`, `joinRelated`, or whatever your tooling calls it) explicitly. Don't trust defaults.

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
| "The ORM defaults are fine" | ORMs default to lazy loading. Eager-load explicitly. |
| "Premature optimization" | N+1 is not optimization — it's correctness. |

## Reference

- Martin Fowler, *Patterns of Enterprise Application Architecture* (2002) — names the pattern and the canonical eager-loading vs. lazy-loading distinction; the "N+1 selects" problem traces to its lazy-load discussion.
- [PostgreSQL EXPLAIN ANALYZE](https://www.postgresql.org/docs/current/sql-explain.html) — use it to inspect the final query plan after batching or joining; use request-level query logs to prove query count dropped from N+1 to O(1).
