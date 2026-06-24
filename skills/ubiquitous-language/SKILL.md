---
name: ubiquitous-language
description: Use when naming a function, type, schema column, route, or any user-facing identifier in a project that has a domain. Use when stakeholders use words for things that the code uses different words for. Use when "what should we call this?" comes up in code review. Use when reading code in a project area you're unfamiliar with and need to map between domain terms and code terms.
---

# Ubiquitous Language

## Overview

**Names in code must use the exact words domain experts and users use for the same concepts.** Variables, types, functions, schemas, routes, error messages, log fields — all of them. When the project's domain vocabulary shifts, the code's vocabulary shifts with it.

There is no "developer dialect" of the domain. If the business says "invoice" and the code says `bill_record`, the code is lying about what it is.

## The Iron Rule

```
NEVER invent code-only names for concepts the domain already names. The domain's words are the code's words.
```

**No exceptions:**
- Not for "I prefer `data`/`obj`/`record` — they're generic and reusable"
- Not for "the domain word is too long"
- Not for "domain experts will rename things again"
- Not for "naming is hard, let's just ship"

## Why

A domain has a vocabulary. *"Consultation."* *"Diagnosis."* *"Pull request."* *"Customer order."* Each word has a precise meaning in the domain — sometimes precise enough that two near-synonyms in everyday English mean different things in the domain. ("Customer" vs. "user" vs. "account holder" — in some businesses, these are distinct entities.)

When developers invent parallel vocabulary, three things happen:

1. **Translation tax.** Every discussion between developers and non-developers has a translation step. Every onboarding requires learning two glossaries.
2. **Domain drift.** When the business pivots, the code lags — because what the business now calls X is still called Y in code. Renaming debt accumulates.
3. **Bug-class via mis-mapping.** A developer maps "user" to the User entity when the business actually meant "customer" (a different entity). The bug surfaces at runtime, in production, in a spreadsheet of weird records.

The fix is structural: **adopt the domain's words and update them when the domain updates them.** Names in code are not for developers; they're for everyone who reads the code over the project's lifetime — including the future you who's forgotten what `proc_v2_holder` was supposed to mean.

This is the central insight of Domain-Driven Design — but it applies to *any* codebase with a real domain, not just enterprise software. A weather app has a domain. A landing page has a domain. The domain may be small, but the words matter.

## Detection

You are violating the rule if any of these are true:

- A type, function, or variable name uses an abbreviation, acronym, or invented term that doesn't appear in the user-facing UI, docs, or business conversations.
- The same concept has different names in different layers (`order` in UI, `purchase_order` in the schema, `OrderRecord` in the type, `purchaseObj` in tests).
- A field name was chosen for "developer convenience" (`obj`, `data`, `info`, `record`, `entity`, `model`) when the domain has a real name.
- A code comment is needed to explain what an identifier means in domain terms (`// this is what the business calls an "invoice"`).
- A PR review surfaces "what's `xyz_handler`?" — and the answer requires reading the body.
- The business pivoted (renamed customers → members, charges → subscriptions) and the code still uses the old terms a month later.

## The Pattern

### Names come from the domain glossary, not the developer's head

Suppose you're building a clinical tool for a medical practice. The clinicians say:
- *consultation* (the visit itself)
- *presenting complaint* (what the patient reports)
- *diagnosis* (the clinician's finding)
- *prescription* (medication and dosage, separately from referrals)
- *follow-up* (scheduled re-visit)

```ts
// ❌ Developer dialect.
type Record = { id: string; complaint: string; outcome: string };
type Visit = { id: string; patient: string };

// ✅ Domain language preserved.
type Consultation = {
  id: ConsultationId;
  presentingComplaint: string;
  diagnosis: Diagnosis | null;
  prescriptions: readonly Prescription[];
  followUpAt: Date | null;
};
```

The key is that the *concept's identity* is preserved — "consultation" is a specific, defined thing in clinical practice, not a generic "visit"; "prescription" is distinct from "referral" even though both come out of the same encounter.

### Match the granularity of distinctions

If the domain distinguishes between "customer" and "account holder" — say a business where multiple users share one account — then the code must distinguish too:

```ts
// ❌ Conflates two domain entities.
type User = {
  id: string;
  email: string;
  billingAddress: Address;
  permissions: Permission[];
};

// ✅ Two domain entities, two types.
type AccountHolder = { // the entity billed and contracted with
  id: AccountHolderId;
  billingAddress: Address;
  email: string;
};

type Customer = { // the individual who logs in and uses the app
  id: CustomerId;
  accountHolderId: AccountHolderId;
  email: string;
  permissions: Permission[];
};
```

The distinction is invisible until it matters — when the billing department asks "who's this *user*?" and a developer has to translate three layers to answer.

### Schema column names follow the same rule

```ts
// ❌ Engineering-speak in the schema.
export const records = pgTable('records', {
  id: uuid('id').primaryKey(),
  obj_name: text('obj_name'),
  start_dt: timestamp('start_dt'),
  end_dt: timestamp('end_dt'),
});

// ✅ Domain terms.
export const consultations = pgTable('consultations', {
  id: uuid('id').primaryKey(),
  presentingComplaint: text('presenting_complaint').notNull(),
  startsAt: timestamp('starts_at').notNull(),
  endsAt: timestamp('ends_at').notNull(),
});
```

The DB column names will appear in monitoring dashboards, support queries, ad-hoc analytics, schema migrations. Domain language there pays off forever.

Casing (`starts_at` vs `startsAt`) is a *separate*, orthogonal convention — follow your stack's norm (Prisma defaults to camelCase columns; many Drizzle/Postgres setups use snake_case). Ubiquitous language is about the *word*, not the case: `starts` over `start_dt`, `presentingComplaint` over `cc`. Both `starts_at` and `startsAt` satisfy this rule; `start_dt` doesn't.

### Routes mirror domain entities, not implementation

```ts
// ❌ Implementation peeks through.
app.get('/api/v1/db/records/consultation/:id', /* ... */);

// ✅ Domain noun + verb.
app.get('/api/consultations/:id', /* ... */);
```

URLs are public, indexed, and quoted in support tickets. They're the most-read identifier in your system.

### Update the language when the domain updates

This is the hard part. When the business renames a concept (customers → members; charges → subscriptions; orders → reservations), the code must follow within days, not months.

The pattern:

1. Update one layer at a time, starting with the most user-visible (UI strings) and ending with the most internal (DB columns).
2. Use migrations (DB rename, code rename) — don't introduce parallel terms.
3. Search the codebase: comments, error messages, log fields, test fixtures. All of them.
4. Update the domain glossary doc (if it exists) so newcomers learn the new term.

If the migration is too painful to do all at once — DB renames are sometimes legitimately gnarly — at minimum, ban the old term from *new* code immediately, and convert the old term in user-facing files where it's visible.

### Domain glossary as a living doc

Maintain a glossary file in the repo (`docs/glossary.md` or similar) that lists every domain term and what it means in *this codebase*. The glossary:

- Lives next to the code.
- Is updated in the same PR that introduces or renames a domain term.
- Defines distinctions ("customer vs. account holder").
- Notes any deliberate divergence from common English ("we say 'order' for what most ecommerce calls a 'cart' until checkout").

The glossary makes onboarding fast and prevents drift; it's also the artifact you point at in code review when someone proposes a name that diverges.

## Pressure Resistance

### "I prefer `data` / `obj` / `record` — they're generic and reusable"

Generic names are *anti-reusable* — they tell readers nothing about what's in the variable. Reusability comes from *the right level of abstraction*, not from naming everything as if it were a `Thing`.

### "The domain word is too long"

Names should be as long as their meaning requires. `presentingComplaint` is two camel-cased words; the meaning is precise. `complaint` is shorter, but now every reader has to ask whether it's the patient's report, the formal incident report, or the support ticket. Net cost is higher.

### "Domain experts will rename things again"

Yes — and the code follows. The cost of one rename across a codebase that uses domain language is small (it's a search-and-replace). The cost of *not* renaming and accumulating two parallel vocabularies is permanent.

### "Naming is hard, let's just ship"

Naming is hard *because it's important*. A name that lasts the project's life is worth ten minutes of thought now. The thought is one of the highest-leverage activities in software engineering.

### "I'll rename later when the business is sure about the term"

The business is rarely "sure" — they refine vocabulary over time. Start with the best term they have today; when it changes, change with them. The agility to rename is the whole point.

## Red Flags

- Variable names ending in `Obj`, `Data`, `Info`, `Record`, `Entity`, `Model` (without a specific domain noun before them).
- Abbreviations or acronyms in identifiers that aren't terms the business uses (`cfg`, `mgr`, `proc`, `usr`).
- A code comment defining what an identifier "represents in business terms."
- A type and a schema table with different names for the same concept.
- A generic verb standing alone or fronting another generic word (`processData`, `handleStuff`, `manageRecords`) where a domain verb would fit (`approve`, `reject`, `archive`, `publish`). A generic verb paired with a domain term is fine, and often correct by layer: `handleCancelBooking` (an orchestration entry point) and a repository's `updateNoShow` / `findById` (persistence CRUD) read clearly — the domain verb belongs in the service layer, the CRUD verb in the repository.
- "Translation" code: a function whose only job is to map between `DbOrder` and `ApiOrder` and `UiOrder` because each layer renamed the same concept.
- A pivot or rename happened in the business >2 weeks ago and the code still uses the old word.

**All of these mean: the code has its own dialect — bring it back to the domain's words.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's just an internal name, users don't see it" | Internal readers do. Internal names become external in logs, errors, support tickets. |
| "Generic names are flexible" | Generic names hide meaning. Specific names are extensible by being clear. |
| "The right name will emerge" | It might — but only if you commit to renaming when it does. "Emerge" without commitment means drift. |
| "Other codebases use these names" | Other codebases have other domains. Your domain has its own vocabulary. |
| "I'd have to update too many files" | Search-and-replace is one command. The translation tax of not doing it lasts forever. |
| "Naming is bikeshedding" | Naming is the single most read-impacting decision in the codebase. It's the opposite of bikeshedding. |

## Related

- `naming-ahclc` — names enforce the domain language
- `comments-as-why-not-what` — comments preserve domain rules

## Reference

- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003) — coined the term **ubiquitous language**. The book's central thesis: a project's domain experts and developers must share a vocabulary, and that vocabulary lives in the code.
- Steve McConnell, *Code Complete* (2nd ed., 2004), ch. 11 ("The Power of Variable Names") — argues the same principle from the construction side: names that read like the problem domain make code self-documenting.
- Robert C. Martin, *Clean Code* (2008), ch. 2 ("Meaningful Names") — "Use Problem Domain Names" as a named rule.
