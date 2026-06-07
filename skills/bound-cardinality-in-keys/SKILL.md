---
name: bound-cardinality-in-keys
description: Use when emitting a metric label, cache key, log dimension, error-monitoring tag, or any string that becomes a "key" in an external system. Use when tempted to include `userId`, `email`, a URL path with params, or any other unbounded user input as a label. Use when reviewing an error-tracker dashboard with millions of unique fingerprints, or a metric-provider bill that doubled overnight.
---

# Bound Cardinality in Keys

## Overview

**Never use unbounded user input as a metric label, cache key prefix, log dimension, error tag, or trace attribute key.** Keys must come from a closed, low-cardinality set — ≤100 distinct values per dimension is a useful upper bound.

User IDs, email addresses, URL paths with parameters, free-text fields — these belong in *attributes* or *bodies*, never in *keys* or *labels*.

## The Iron Rule

```
NEVER put unbounded user input into a metric label, error tag, cache key, or span name.
```

**No exceptions:**
- Not for "I need to see per-user metrics"
- Not for "just this one label"
- Not for "our budget can handle it"
- Not for "it's only in development"

## Why

External observability and caching systems treat each unique key combination as a separate time series, group, or entry. Sending a metric with `user_id` as a label creates one time series per user — millions of them, each with sparse data, each indexed and stored.

The consequences:

- **Bill shock.** Metric providers and error trackers bill on cardinality. An unbounded label can 10–100× your monthly bill in days.
- **Performance degradation.** Aggregation queries slow to a crawl when grouping millions of series.
- **Dashboard breakage.** UI dropdowns trying to enumerate the label become unusable.
- **Loss of alerting.** Per-series alerts can't be defined when the series space is open.
- **Cache pollution.** A cache key built from user input gets one entry per unique input — most entries are never read again.

The fix is architectural: **separate dimensions (low-cardinality, queryable) from attributes (high-cardinality, observable but not aggregated)**. Different storage shapes for different needs.

## Detection

You are violating the rule if any of these are true:

- A metric/counter `.inc({ user_id: '...' })` or `.inc({ email: '...' })`.
- An error tag `tracker.setTag('userId', user.id)` (tags are indexed; use contexts for high-cardinality data).
- A cache key built from a free-text query string: `` cache.get(`search:${q}`) ``.
- A log field used as a dashboard "group by" with unbounded values (e.g., `path: '/users/abc-123-uuid'`).
- A counter with a `request_id` label.
- A `histogram(name, value, { tag: someRawHeader })`.
- An OpenTelemetry span attribute key (not value) derived from input.

## The Pattern

### Metrics — label by category, not identity

```ts
// ❌ One time series per user. Cardinality = number of users.
counter('api.requests', 1, { userId: req.userId, path: req.url });

// ✅ Time series per route × status. Cardinality = routes × statuses ≈ dozens.
counter('api.requests', 1, {
  route: '/api/invoices/[id]',  // template, not the actual path
  method: 'GET',
  status_class: '2xx',          // bucket, not the raw status
});
```

The route is a *template* (`/api/invoices/[id]`), not the rendered URL (`/api/invoices/abc-123`). Status is a *bucket* (`2xx`, `4xx`, `5xx`), not the exact code (which is also low-card, ~30 values, also fine).

User-level data lives in *logs* and *traces*, where the system is designed for unbounded attribute values.

### Errors — tag vs. context

Most error trackers index *tags* (creating one series per unique value) but store *contexts* on the event itself (not indexed). Use the distinction:

```ts
// ❌ Tags are indexed. setTag with user.id = millions of fingerprints.
tracker.setTag('user_id', user.id);
tracker.setTag('org_id', org.id);

// ✅ Tags = low-cardinality. Use plan tier, feature flag value, region.
tracker.setTag('plan', user.plan);     // 'free' | 'pro' | 'enterprise' — fine
tracker.setTag('region', user.region); // 'us-east' | 'eu-west' — fine

// ✅ Use contexts / user-object for high-cardinality data — visible in the event, NOT indexed.
tracker.setUser({ id: user.id, email: user.email });
tracker.setContext('order', { id: order.id, amount: order.amountCents });
```

The user-object and context fields store data on the *event* but don't create indexed dimensions. They're searchable but don't multiply cardinality.

### Logs — structured fields are fine, but think about what's groupable

Log aggregators typically allow unbounded attribute values. The cardinality concern is *which fields end up as facets* in dashboards.

```ts
log.info({
  event: 'invoice_paid',         // ← low-card enum, this is the "type" key
  invoice_id: invoice.id,        // ← high-card attribute, fine for search
  amount_cents: invoice.amountCents,
  user_id: user.id,              // ← high-card, also fine for search
  plan: user.plan,               // ← low-card, can be faceted
});
```

The `event` field is the equivalent of a metric name — *that's* what gets faceted/grouped. User IDs and invoice IDs are searchable, not grouped.

### Cache keys — only by deterministic, low-card axes

```ts
// ❌ Free-text query in the cache key. One entry per unique search.
const result = await cache.get(`search:${q}`);

// ✅ Cache only by canonical, low-card identity.
const result = await cache.get(`invoice:${invoiceId}`);

// For free-text searches: don't cache by query, cache the underlying lookups.
// Or, if caching searches is genuinely valuable: hash the normalized query
// AND set a short TTL so unused entries fall out fast.
const normalized = q.toLowerCase().trim();
const result = await cache.get(`search:${hash(normalized)}`, { ttl: 60 });
```

The hash variant bounds the *key length* but not the *number of keys* — make sure the TTL is short enough that low-hit keys evict.

### Tracing — span names from a closed set

```ts
// ❌ Span name from input.
tracer.startSpan(`GET ${req.url}`);

// ✅ Span name = route template; attributes carry the specifics.
tracer.startSpan('GET /api/invoices/[id]', {
  attributes: {
    'http.url': req.url,           // attribute = high-card, fine
    'http.invoice_id': invoiceId,
  },
});
```

OpenTelemetry semantic conventions distinguish *operation name* (low-card, used for grouping) from *attributes* (any cardinality).

### Cardinality budget — name it

For each metric, declare its label set explicitly:

```ts
// metrics/labels.ts
export const requestLabels = ['route', 'method', 'status_class'] as const;
//                              ~50         ~5       3
//                              → cardinality budget: ~750 series for this metric
```

When someone adds a label, they update this file. The cardinality multiplication is visible.

## Pressure Resistance

### "I need to see per-user metrics"

You need to *query* per-user — that's a different system. Use logs or traces, where per-user lookup is supported. Metrics are for aggregates ("how many requests per route in the last hour"), not for "what did user X do."

### "Just this one label"

The cost of one label propagates. The next developer adds one. The one after, another. Three "just this one" labels turn 1000 series into 1,000,000. Don't seed the precedent.

### "Our budget can handle it"

Today, maybe. Cardinality grows with the user base; budgets grow with revenue. The two race, and cardinality usually wins. Cap it at the source.

### "We need to alert per-user"

Alerts per-user belong in *behavioral* systems (fraud detection, support escalation), not metric systems. The right tool is a log-based alert ("error rate for user X exceeded threshold"), which doesn't blow up metric cardinality.

### "I'll add it and remove if the bill goes up"

The bill goes up *retroactively* — you pay for the high-cardinality month. Add labels conservatively; subtraction doesn't refund.

### "It's only in development"

Development data flows through the same pipelines and contributes to the same indexes. The bill is one environment, but the architectural mistake is recorded everywhere.

## Red Flags

- A metric label whose value is `req.userId`, `req.email`, `req.url`, `req.headers['x-*']`, or any unmodified user input.
- An error-tracker `setTag` with user-supplied or per-request data.
- A cache key constructed by template-string concatenation with input.
- A monitoring dashboard with a `group by` dropdown that takes 30 seconds to load.
- A monthly metrics bill that doubled with no corresponding traffic change.
- A code review comment "let's just add `userId` as a label."
- The phrase "we need this dimension for filtering" — filtering is search, not aggregation.

**All of these mean: unbounded input is leaking into a key — move it to attributes or bodies.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Tagging users helps with debugging" | Tag with low-card categories; put user IDs in trace/log attributes. |
| "We don't have that many users" | You will. By the time you do, the cardinality is baked in. |
| "Hashing the user ID bounds the cardinality" | Hashing a high-card value produces a different high-card value. Bucketing (`hash % 100`) does work — but that's a coarse aggregate, which is what categories are. |
| "OpenTelemetry handles this" | OTel exports your labels as-is. The collector won't save you from cardinality you generated. |
| "We pay for unlimited cardinality" | No vendor offers truly unlimited; even high-cardinality observability tools have practical limits. And the queries slow down before the bill does. |
| "It's the only way to find specific events" | Use logs or traces, which are designed for it. Metrics aren't. |

## Reference

- Charity Majors, Liz Fong-Jones, George Miranda, *Observability Engineering* (O'Reilly, 2022) — chapters on cardinality and the distinction between metrics, logs, and traces. The core insight: *observability is high-cardinality by design; metrics are aggregate by design; mixing the two ends badly.*
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/) — the formal distinction between *names* (low-card, for grouping) and *attributes* (any cardinality, for context).
