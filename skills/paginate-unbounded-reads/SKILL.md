---
name: paginate-unbounded-reads
description: Use when a query has no `LIMIT` and the table grows with usage. Use when a list endpoint returns every matching row. Use when a `findMany()` has no `take`/cursor. Use when an endpoint that was instant in dev times out in production months later. Use when an export or batch job loads an entire table into memory. Use when a client renders "all" of anything user-generated.
---

# Paginate Unbounded Reads

## Overview

**A read whose result size grows with usage ships with an explicit bound — a `LIMIT`, a page with an enforced maximum, or a bounded stream window.** The query that returns 50 rows today returns 500,000 after two years of success; nothing in its code changed, only the data underneath it.

This is one of three sibling resource bounds: `n-plus-one-prevention` bounds how many queries a request makes, `cap-async-fan-out` bounds how many run at once, and this bounds how much a single query returns.

## The Iron Rule

```
NEVER ship a read whose result size grows with usage without an explicit bound.
```

**No exceptions:**
- Not for "there isn't much data yet"
- Not for "it's an internal admin page"
- Not for "the client needs everything for the filter/search UI"

**The scope clause does the work.** Sets bounded *by the domain* — a user's payment methods, the enum-sized `plans` table, line items on one invoice — don't grow with usage; the bound exists by construction, and paginating them is ceremony. The rule triggers when cardinality tracks success: users, orders, events, messages, logs.

## Why

Michael Nygard catalogs this as the **Unbounded Result Sets** stability antipattern: the application trusts the database not to return too much, and the database — asked for everything — cheerfully complies. The failure arrives on three axes at once:

1. **Memory.** Rows × row width × the copies your stack makes (driver buffer, ORM objects, JSON serialization) — a full-table read materializes all of it simultaneously. The OOM kills the whole process, not just the greedy request.
2. **Latency.** The endpoint's response time is now a function of table size. It degrades monotonically, slowly enough that no deploy is to blame, until it crosses the timeout and becomes an outage.
3. **Everything else.** A long scan holds a pooled connection and floods the network; concurrent requests queue behind it — one unbounded read is a self-inflicted load spike (`shed-load-under-overload`'s problem, caused from inside).

```ts
// ❌ Result size = however many orders this org ever accumulates.
export async function listOrders(orgId: OrgId) {
  return db.query.orders.findMany({ where: eq(orders.orgId, orgId) });
}

// ✅ Bounded: default page size, enforced max, and a cursor for the rest.
const listParams = z.object({
  limit: z.coerce.number().int().min(1).max(100).default(25), // clamp — never trust
  cursor: z.string().optional(),
});
```

The clamp matters as much as the default: a `limit` the client controls without a server-side max is still an unbounded read — it just requires the attacker (or the over-eager frontend) to ask.

## Detection

You are violating the rule if any of these are true:

- A `findMany` / `SELECT` with no `take` / `LIMIT` on a table whose row count tracks usage.
- A list endpoint whose response size nobody can state an upper bound for.
- A client-controlled `limit`/`pageSize` parameter with no server-side maximum.
- An export/report/batch job that loads a whole table with one query instead of streaming or looping in batches.
- `.map`/`.filter`/`.reduce` in memory over "all rows" to compute something the database could aggregate.
- A list endpoint that is noticeably slower in production than staging — data-size-dependent latency is this smell in motion.

## The Pattern

### Bound the API: default, maximum, and a signal for more

The convention Stripe's API popularized — a modest default, a hard max, and an explicit `has_more` — is the shape to copy. The response tells clients how to continue so they never resort to `limit=100000`.

```ts
export async function listOrders(orgId: OrgId, raw: unknown) {
  const { limit, cursor } = listParams.parse(raw);
  const page = await db.query.orders.findMany({
    where: and(eq(orders.orgId, orgId), cursor ? lt(orders.id, decodeCursor(cursor)) : undefined),
    orderBy: desc(orders.id),
    limit: limit + 1, // fetch one extra to learn whether more exist
  });
  const hasMore = page.length > limit;
  const items = hasMore ? page.slice(0, limit) : page;
  const last = items[items.length - 1];
  return {
    items,
    hasMore,
    nextCursor: hasMore && last ? encodeCursor(last.id) : null,
  };
}
```

### Cursor/keyset over offset for anything deep

`OFFSET n` *scans and discards* n rows — page 10,000 costs 10,000 rows of work, and rows inserted mid-pagination shift every subsequent page (skipped or duplicated items). Keyset pagination (`WHERE id < $cursor ORDER BY id DESC LIMIT k`) does index-seek work per page regardless of depth and is stable under concurrent writes. Offset is acceptable for shallow, human-paged UIs; anything a machine iterates gets a cursor.

### Internal reads count too: batch or stream, never "load all"

The rule isn't only about HTTP. A backfill, export, or nightly job bounds its working set the same way — a keyset loop, or a driver-level stream with a bounded window (the same bounded-batch discipline `steady-state-purge-unbounded-growth` applies to deletes):

```ts
// ✅ Keyset loop: bounded memory no matter how large the table is.
let cursor: OrderId | undefined;
for (;;) {
  const batch = await db.query.orders.findMany({
    where: cursor ? gt(orders.id, cursor) : undefined,
    orderBy: asc(orders.id),
    limit: 1000,
  });
  if (batch.length === 0) break;
  await exportBatch(batch);
  cursor = batch[batch.length - 1]!.id;
}
```

### Aggregate in the database, not in process

If the unbounded read exists to compute a count, a sum, or a "latest per group", the bound isn't pagination — it's pushing the aggregation into SQL (`COUNT`, `SUM`, `DISTINCT ON`, window functions) so the result set is small by definition.

## Pressure Resistance

### "There isn't much data yet"

That's precisely the condition under which unbounded reads get written — they only *stay* safe if the product fails. The bound costs a few lines now; retrofitting pagination onto a shipped API contract later costs a versioned API-contract migration, with every existing consumer to carry through it.

### "The client needs the full list for its search/filter UI"

Then the requirement is *search*, not "ship the table." Server-side filtering with a bounded result, or a search index, serves the UI without making response size a function of table size. "Send everything, filter in the browser" is the same unbounded read wearing a UX justification.

### "It's an internal admin tool"

Admin tools query the *biggest* aggregates (all users, all orders) with the least review. The OOM doesn't check who was asking; the process it kills serves customers too.

### "Pagination complicates the client"

A `nextCursor` loop is a dozen lines, written once per client. The alternative complexity — timeouts, OOMs, and a page that renders 200k DOM nodes — is unbounded.

## Red Flags

- `findMany()` with no `take` on a usage-growing table.
- `SELECT ... WHERE org_id = $1` with no `LIMIT` in a request path.
- A client-supplied page size used without clamping.
- `OFFSET` arithmetic in anything a machine iterates.
- An in-memory `reduce` over a full table to produce one number.
- "Load all, filter client-side" as a feature's data strategy.

**All of these mean: state the bound — clamp the page, cursor the rest, aggregate in the database.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's fast in staging" | Staging has staging's data. The read's cost is a function of production's. |
| "We'll paginate when it's slow" | "Slow" arrives as a production timeout, and the fix then is an API contract change. |
| "The ORM streams it, probably" | Most ORMs materialize the full result by default. Streaming is an explicit choice — make it. |
| "Our biggest customer only has 2k rows" | Your biggest customer is your fastest-growing table. Success is the load test. |
| "A max limit will break power users" | A power user hitting the max is the pagination working — that's the signal to iterate pages, not to remove the ceiling. |

## Related

- `n-plus-one-prevention` — the sibling bound: that skill caps query *count* per request; this caps result *size* per query.
- `steady-state-purge-unbounded-growth` — that skill bounds what you *store*; this bounds what you *read*. Its batched deletes and the export loop above share the same bounded-batch discipline.
- `shed-load-under-overload` — an unbounded read is overload you inflicted on yourself; bounding reads is upstream of shedding.
- `parse-dont-validate` — the page params (`limit`, `cursor`) are boundary input: parse and clamp them, never trust a raw `limit`.
- `cap-async-fan-out` — the third resource bound in the family: result size, query count, and fan-out concurrency.

## Reference

- Michael Nygard, *Release It!* (2nd ed., 2018) — the **Unbounded Result Sets** stability antipattern: trusting the other system to return a reasonable amount lets it dictate your memory and latency; the remedy he prescribes is limits and pagination in the application-level protocol.
- [Stripe API — Pagination](https://docs.stripe.com/api/pagination) — the widely-copied convention: default 10, max 100, `has_more`, cursor parameters.
- No single canonical spec exists for pagination — the rule is engineering consensus (Nygard's antipattern + the cursor convention major APIs converged on), not an RFC. Treat the numbers as conventions to tune, the bound itself as the requirement.
