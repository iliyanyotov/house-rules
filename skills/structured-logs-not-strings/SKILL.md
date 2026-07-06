---
name: structured-logs-not-strings
description: "Use when writing a log call that concatenates or interpolates variables into the message string, or when `console.log` appears in service code. Use when an incident requires grepping prose with a regex to find all failures for one order. Use when an alert can't be defined because the error code lives inside a sentence. Use when the same fact is logged as `order 123 failed`, `failed order 123`, and `Order#123 failure` in three different files."
---

# Structured Logs, Not Strings

## Overview

**Every log line a service emits is a structured event: a constant message naming what happened, plus a fields object carrying every variable.** `log.error('order_fulfillment_failed', { orderId, errorCode })`, never `console.log('order ' + orderId + ' failed')`.

Prose is for humans reading one line; production logs are queried, grouped, joined, and alerted on by machines reading millions. Data that exists only inside a sentence is data you can't query.

Scope: logs a service ships to an aggregator — including one-off scripts run against production, whose output is the audit trail. Human-facing CLI/TUI output is presentation, not logging, and is out of scope.

## The Iron Rule

```
NEVER emit a production log line whose variable data exists only inside an
interpolated prose string. Every variable is a field on a structured event.
```

**No exceptions:**

- Not for "it's a temporary debug line" — temporary lines ship, and then page people
- Not for errors — the error object is a field, not a string suffix
- Not for scripts and workers — anything that runs against production emits logs, not prints
- Local pretty-printing is not an exception: it's a rendering transport over the same structured call site

Within the rule, one strong _default_ rather than invariant: keep the message itself constant. Duplicating a field into the message for readability (`` `invoice ${id} sent` `` with `invoiceId` also in fields) doesn't lose data, but it defeats grouping — a constant message is what lets the aggregator count "how many of _these_". Prefer constant.

## Why

The concrete failure: an incident is open, payments are failing for one customer, and the on-call engineer needs "every log line about order 4f2a… in the last hour". With structured logs that's one field filter. With prose it's a regex archaeology session across `order 4f2a failed`, `failed to fulfill order: 4f2a`, and `Order 4f2a — fulfillment error`, and the engineer _still_ can't be sure the regex caught every variant. The same shape blocks alerting: "page when `errorCode=card_declined` exceeds 100/min" is trivial on a field and impossible on free text.

Structured logs are also the substrate everything else stands on: request-scoped correlation, log-based metrics, joining a user's journey across services. A string log can't grow into any of those; a structured one already has.

## Detection

You are violating the rule if any of these are true:

- `console.log` / `console.error` in service code instead of the logger.
- A log call whose message is built with `+` or a template literal around variables, with no fields object.
- `String(err)` or `err.message` concatenated into a message — the stack, `code`, and `cause` are gone.
- The same event logged with differently-worded messages or differently-named fields in different files.
- A dashboard query that starts with a regex over the message body.
- An alert request that can't be built because the discriminating value is inside prose.

## The Pattern

### Constant event name + fields

```ts
try {
  await fulfillOrder(orderId);
} catch (err) {
  // ❌ Prose. Unqueryable, ungroupable, stack and error code destroyed.
  console.log(
    "order " + orderId + " failed for " + userId + ": " + String(err),
  );

  // ✅ Constant event name; every variable is a field.
  log.error("order_fulfillment_failed", {
    orderId,
    userId,
    err: serializeError(err),
  });
  throw err;
}
```

The message is the event's _name_ — a low-cardinality constant, exactly like a metric name. The fields carry the specifics.

```ts
// ❌ The variable is in a field — but the message mutates per invoice, so nothing groups.
log.info(`invoice ${invoiceId} sent`, { invoiceId });

// ✅ One message, one group, one count.
log.info("invoice_sent", { invoiceId });
```

### Errors are fields, serialized whole

`String(err)` flattens an error to `"Error: connect ECONNREFUSED"` — no stack, no `code`, no `cause` chain. Serialize the object into a field (`serializeError` → `{ name, message, stack, code }`) so the aggregator can facet on `err.name` and the human can read the stack. If your typed errors carry a `code` field, that code is the thing to alert on — surface it as its own top-level field too.

### One vocabulary for field names

A field is only queryable if it has the _same name everywhere_. Pick one convention and hold it:

| Wrong (three files, three names) | Right (everywhere)                                   |
| -------------------------------- | ---------------------------------------------------- |
| `orderId`, `order_id`, `oid`     | `orderId`                                            |
| `error`, `err_msg`, `e`          | `err` (serialized object), `errorCode` (string code) |
| `duration`, `elapsed`, `time`    | `durationMs` — unit in the name                      |
| `user`, `uid`, `userId`          | `userId`                                             |

A shared constants module or a typed logger wrapper makes the convention checkable instead of aspirational.

### Request-scoped context — bind once, inherit everywhere

Fields that apply to every line of a request (`requestId`, `userId`) are bound once on a child logger, not repeated at every call site:

```ts
export async function handlePayInvoice(req: Request): Promise<Response> {
  const requestId = req.headers.get("x-request-id") ?? crypto.randomUUID();
  const reqLog = log.child({ requestId });

  const body = PayInvoice.parse(await req.json());
  reqLog.info("invoice_payment_started", { invoiceId: body.invoiceId });

  const result = await payInvoice(body, { log: reqLog });
  reqLog.info("invoice_payment_completed", {
    invoiceId: body.invoiceId,
    paymentId: result.paymentId,
  });
  return json({ paymentId: result.paymentId }, { status: 200 });
}
```

Now every line of the request carries `requestId`, and one filter reconstructs the whole request's story. This binding is also the anchor for cross-service correlation: propagate the same ID on outbound calls and the story spans services.

### Levels mean something

| Level   | Meaning                                   | Example                                        |
| ------- | ----------------------------------------- | ---------------------------------------------- |
| `error` | Broken and needs action                   | unhandled failure, invariant violated          |
| `warn`  | Degraded but handled                      | fallback taken, retry exhausted, dead-lettered |
| `info`  | A state change worth an audit trail       | `invoice_sent`, `subscription_canceled`        |
| `debug` | Diagnostic detail, off by default in prod | payload shapes, branch decisions               |

A codebase where everything is `info` has no levels; alerts on `error` are only as trustworthy as the discipline that keeps non-errors out of `error`.

### Fields vs. metric labels — different cardinality rules

```ts
// Log field: unbounded values are fine — logs are for search.
log.info("payment_captured", { paymentId, userId });

// Metric label: bounded values only — metrics are for aggregation.
counter("payments.captured", 1, { method: "card", status_class: "2xx" });
```

A `userId` belongs in a log field and never in a metric label. The full rule — and what happens when you get it wrong — is `bound-cardinality-in-keys`; the short version is that log fields are searched while metric labels are indexed as permanent time series.

### What never goes in a field

Structure makes logs _more_ leak-prone, not less — a fields object invites dumping whole entities, headers, and request bodies, secrets included. `secrets-handling` owns the never-log list (credentials, tokens, signatures, card data) and the redaction discipline; apply it to every fields object, and log projections (`{ userId, plan }`), not entire objects.

## Pressure Resistance

### "It's just a quick debug line"

The quick debug line is exactly the one that fires during the next incident, in prose, at volume. Writing it structured costs five extra characters; the logger is already imported.

### "Prose is more readable"

For one line on your terminal, yes — which is what a pretty-print transport in development is for. In the aggregator, "readable" means _filterable_: ten thousand prose lines are unreadable in a way ten thousand structured events are not.

### "We can always parse the strings later"

Post-hoc parsing means writing and maintaining a regex per message variant, forever, and every reworded message silently breaks a dashboard. Emitting structure at the source is strictly cheaper than reconstructing it downstream.

### "console.log works fine"

`console.log` has no levels, no fields, no child context, no redaction hook, and writes unstructured text. Every property you need later is missing. The logger is one import away.

### "We'll standardize field names when we adopt tracing"

Tracing _assumes_ the discipline: consistent IDs in consistent fields. Adopting it on top of prose logs means doing this migration anyway, at incident-count interest rates.

## Red Flags

- `console.log` in a service or worker.
- `+` or a bare template literal building a log message with no fields object.
- `String(err)`, `err.message`, or `JSON.stringify(err)` inside a message string.
- The same field under three names in three files.
- A saved dashboard query containing a regex over message text.
- A fields object containing a whole request, whole user, or anything `secrets-handling` bans.
- `log.info` used for failures because "it shows up either way".

**All of these mean: data is trapped in prose — name the event, move every variable into a field.**

## Common Rationalizations

| Excuse                                     | Reality                                                                       |
| ------------------------------------------ | ----------------------------------------------------------------------------- |
| "Prose reads better"                       | On one line, locally — use a pretty transport. The aggregator reads fields.   |
| "We'll parse it later"                     | A regex per message variant, broken by every reword. Structure at the source. |
| "It's temporary"                           | Temporary log lines have the longest half-life in any codebase.               |
| "The message already says it"              | Says it to a human, once. A field says it to every query and alert, forever.  |
| "Structured logging is a library decision" | It's a call-site decision. Any logger does it; the discipline is yours.       |
| "We log so little it doesn't matter"       | Low volume makes each line _more_ load-bearing during an incident, not less.  |

## Related

- `secrets-handling` — owns what must never appear in a log line; every fields object is subject to it.
- `bound-cardinality-in-keys` — log fields tolerate unbounded values, metric labels don't; the constant event name is this skill's low-cardinality key.
- `swallow-deliberately-at-the-boundary` — a deliberate swallow is only deliberate if it emits a structured event; a silent catch and a prose catch are both invisible.

## Reference

- OWASP, [_Logging Cheat Sheet_](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) — which events to log, which data to exclude, and the case for a consistent, machine-parseable format.
- Charity Majors, Liz Fong-Jones, George Miranda, _Observability Engineering_ (O'Reilly, 2022) — the structured, wide event as the fundamental unit of telemetry.
- Google, _The Site Reliability Workbook_ (O'Reilly, 2018), ch. 4 ([*Monitoring*](https://sre.google/workbook/monitoring/)) — structured event logging and logs-based metrics as first-class monitoring signals.
