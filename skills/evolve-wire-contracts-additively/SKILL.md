---
name: evolve-wire-contracts-additively
description: Use when changing the shape of an API response, webhook body, or event payload consumed by a party whose deploys you don't control — removing, renaming, or re-typing a field. Use when a consumer schema uses `.strict()` or a closed `z.enum` on a payload another party produces, and a new field or variant starts rejecting every message. Use when "we checked, nobody uses that field" justifies a removal. Use when a partner integration breaks after a deploy that "only cleaned up the API".
---

# Evolve wire contracts additively

## Overview

**A published wire contract — API response, webhook body, event payload — changes additively: add fields, never remove, rename, or re-type in place; a true break gets a new version.** And consumers hold up the other half: parse only the fields you need, tolerate the ones you don't know.

"Published" means consumed by a party whose deploys you don't control — a partner, a mobile build in the wild, another team's service, webhook receivers. Within that scope the rule is an invariant; an interface where you own both sides and both deploys is governed by the cheaper expand/contract discipline instead (see `## Related`). A contract explicitly documented as unstable/preview is also out of scope — that documentation is the one honest way to reserve the right to break.

## The iron rule

```
NEVER remove, rename, or re-type a field in place in a published contract.
Producers change additively and version true breaks; consumers parse what
they need and ignore the rest.
```

**No exceptions:**
- Not for "nobody uses that field" — you can't see the partner cron or the two-year-old mobile build
- Not for "we'll coordinate the deploy with the consumer team" — coordination doesn't cross a party boundary, and old clients linger for years
- Not for "it's cleaner" — cleanliness is what the next version is for

## Why

Your own database migration has a rollover window of minutes that you can wait out. A wire contract has no window: the consumer upgrades when the partner's team gets around to it, when the user updates the app, or never. Every in-place removal or rename is an outage you schedule for someone else, at a time you don't choose, in a codebase you can't see.

The re-type is the nastiest variant because nothing visibly fails. Change `amount` from dollars to cents in place and every consumer still parses a number — and now charges, invoices, or reports are silently wrong by 100×. A removal at least breaks loudly; a re-type corrupts quietly.

## Detection

You are violating the rule if any of these are true:

- A PR deletes or renames a JSON field in a serializer for a public endpoint, webhook, or event, with no new version alongside.
- A field keeps its name but changes meaning or unit (`amount` dollars → cents, `date` local → UTC, string → number).
- A required response field becomes absent-when-null "for cleanliness" — absence is a removal to any consumer that read it.
- A consumer schema calls `.strict()` / `z.strictObject` on a payload another party produces.
- A consumer parses another party's status field with a closed `z.enum` and no fallback — the producer's next additive variant rejects every message.
- The rollout plan for a contract change is "announce it in the changelog and email the partners".

## The pattern

### Producer — add beside, never mutate

A rename is an add. The old field stays, populated, deprecated in docs — not deleted:

```ts
// ❌ v1 shipped { displayName }; this "cleanup" re-ships the same endpoint with { fullName }.
//    Every consumer reading displayName breaks the moment this deploys.
return json({ id: invoice.id, fullName: customer.fullName }, { status: 200 });

// ✅ Additive: the new field beside the old, both populated from the same source.
return json({
  id: invoice.id,
  displayName: customer.fullName, // deprecated 2026-07: prefer fullName
  fullName: customer.fullName,
  amountCents: invoice.amountCents,
}, { status: 200 });
```

This is the producer's dual-write, same instinct as an expand-phase schema change — except the "old readers" are other parties, so the contract phase may never arrive. Budget for the old field living indefinitely, or for retiring it via an explicit version.

### Producer — a re-type is a new field

```ts
// ❌ Same field, new unit. Parses fine everywhere; every consumer is now wrong by 100×.
return json({ id: payment.id, amount: payment.amountCents }, { status: 200 });

// ✅ New meaning, new name. The old field keeps its old meaning until its version dies.
return json({
  id: payment.id,
  amount: centsToDollars(payment.amountCents), // deprecated: prefer amountCents
  amountCents: payment.amountCents,
}, { status: 200 });
```

The rule of thumb: if old consumer code would still typecheck but compute wrong results, the change is a re-type and needs a new name.

### Producer — version true breaks, don't smuggle them

Some changes can't be additive: a field's semantics must change, a resource is restructured, an event's shape is wrong at the root. Those get a *new name*, published alongside the old until the old one's consumers are gone (which may be never):

- Events: `invoice.paid.v2` published alongside `invoice.paid`, each with its own schema.
- HTTP: `/v2/invoices` alongside `/v1/invoices`, with a sunset policy you actually enforce and monitor.
- Consumption metrics per version tell you when — whether — retirement is possible; guessing does not.

The way out of "nobody uses that field" guesswork is consumer-driven contracts: consumers publish tests describing the fields they actually read, and producer CI runs them. Then a removal is provably safe or provably not, instead of argued about.

### Consumer — read tolerantly: take what you need, ignore the rest

```ts
// ❌ Strict parse of the whole payload — the producer's next additive field rejects every event.
const InvoicePaidEvent = z.strictObject({
  eventId: z.uuid(),
  invoiceId: z.uuid(),
  amountCents: z.int().nonnegative(),
  status: z.enum(['open', 'paid', 'refunded']), // their next status value breaks you too
});

// ✅ Parse only the fields you read; unknown keys are stripped (z.object's default —
//    which also enforces "don't re-export their contract" mechanically);
//    unknown *string* variants collapse into a named sentinel. A malformed value
//    (42, null, {}) still rejects — tolerance is for the unknown, not the malformed.
const InvoiceStatus = z.union([
  z.enum(['open', 'paid', 'refunded']),
  z.string().transform(() => 'unrecognized' as const),
]);
const InvoicePaidEvent = z.object({
  eventId: z.uuid(),
  invoiceId: z.uuid(),
  amountCents: z.int().nonnegative(),
  status: InvoiceStatus,
});
```

Tolerance is *asymmetric*: the fields you depend on are parsed strictly — wrong type, missing value, out-of-range all reject, exactly as `parse-dont-validate` demands. The tolerance applies only to fields you don't read and variants you don't know. "Liberal in what you accept" means indifferent to the unknown, never accepting of the malformed.

### Consumer — unknown variants are a named case, not a crash

The sentinel keeps `exhaustive-switch` intact — you switch over the *parsed* union, which includes `'unrecognized'` as a first-class case:

```ts
switch (event.status) {
  case 'open':
  case 'paid':
    return applyPayment(event);
  case 'refunded':
    return applyRefund(event);
  case 'unrecognized':
    // The producer added a variant we don't handle yet. Tolerate loudly: log, count, skip.
    log.warn('unrecognized_invoice_status', { eventId: event.eventId });
    counter('events.unrecognized_variant', 1, { topic: 'invoice.paid' });
    return;
  default: {
    const exhausted: never = event.status;
    throw new Error(`unreachable status: ${JSON.stringify(exhausted)}`);
  }
}
```

Skipping silently would hide a growing blind spot — the log and counter make "we're ignoring 4% of events" visible before it becomes an incident.

### Consumer — don't re-export their contract as yours

Persist and forward only the fields you consumed, not the whole payload. Passing the raw payload through to your own consumers couples *them* to a contract you don't control, and turns you into an unversioned proxy of someone else's API.

## Pressure resistance

### "Nobody uses that field"

You know your consumers' *traffic*, at best — not their code, their crons, or the app versions still installed. Absence of evidence in last week's logs is not absence of a partner's quarterly billing job. If you need this to be provable, that's what consumer-driven contract tests are for.

### "We'll announce it and give everyone 30 days"

Announcements don't deploy other people's code. Some consumers will migrate; the long tail won't, and the long tail includes the integration whose failure becomes your escalation. Additive change needs no migration; that's the point.

### "Versioning is overhead"

Maintaining `v1` beside `v2` is visible, budgeted overhead. An unversioned break is the same cost paid as partner outages, support escalations, and emergency rollbacks — plus the trust. Version when you must break; the overhead was always there.

### "Our OpenAPI spec is the contract — we regenerate the clients"

You regenerate *your* clients. The partner's hand-rolled integration, the mobile build from last year, and the no-code tool reading your webhook regenerate nothing. The spec documents the contract; it doesn't deploy its consumers.

### "Additive-forever means the payload fills with cruft"

It does accumulate — deprecated fields are the rent on a boundary you don't control. The pressure valve is versioning with real sunset enforcement, not in-place deletion. Cruft is ugly; a partner outage is expensive.

## Red flags

- A serializer diff for a public endpoint that deletes or renames a key.
- A field whose unit or semantics changed while its name stayed.
- `.strict()` or `z.strictObject` on a payload from another party.
- A closed `z.enum` on another party's discriminator with no sentinel fallback for unknown variants.
- A `switch` on an external event type whose `default` throws.
- "Cleanup" or "consistency" as the stated motivation for a public-contract change.
- A webhook payload forwarded wholesale into your own events or database.

**All of these mean: a break is crossing a boundary you don't control — make it additive, or give it a version.**

## Common rationalizations

| Excuse | Reality |
|---|---|
| "Nobody uses it" | You can't see their code. Prove it with consumer-driven contracts or don't claim it. |
| "We'll coordinate the rollout" | Coordinated deploys don't cross party boundaries; old app builds linger for years. |
| "It still parses, so it's compatible" | A re-type that parses is the worst break: silently wrong by 100×. |
| "Strict consumer schemas are safer" | Strict on what you read, yes. Strict on the whole payload means their additive change is your outage. |
| "The changelog warned them" | A changelog is not a deploy. Additive changes need neither. |
| "We'll clean it up in place, it's internal-ish" | If any consumer deploys on someone else's schedule, it's published. Treat it so. |

## Related

- `expand-contract-schema-migration` — the sibling discipline for your *own* database: there you control both sides and the overlap window is minutes, so the contract step actually arrives. Across a party boundary the window is unbounded and contract may never come — additive is the steady state, and an explicit new version is the only sanctioned break.
- `parse-dont-validate` — the tolerant reader is a boundary schema: strict on the fields you read, indifferent to the ones you don't.
- `exhaustive-switch` — reconciliation: exhaustiveness applies to the *parsed* union including the `'unrecognized'` sentinel, never to the raw wire value another party can extend.

## Reference

- Martin Fowler, [*TolerantReader*](https://martinfowler.com/bliki/TolerantReader.html) (bliki, 2011) — the consumer half: extract only what you need, ignore the rest.
- Ian Robinson, [*Consumer-Driven Contracts: A Service Evolution Pattern*](https://martinfowler.com/articles/consumerDrivenContracts.html) (martinfowler.com, 2006) — making "who reads what" explicit and testable, so producers can evolve with evidence.
- RFC 1122, §1.2.2 — the canonical restatement of the robustness principle ("be liberal in what you accept, and conservative in what you send"); Postel's original formulation is RFC 761 §2.10 (1980).
- [RFC 9413, *Maintaining Robust Protocols*](https://www.rfc-editor.org/rfc/rfc9413.html) (IAB, 2023) — the modern counterweight: robustness needs active maintenance, and tolerance that conceals peer defects lets them ossify.
