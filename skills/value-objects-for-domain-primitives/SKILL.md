---
name: value-objects-for-domain-primitives
description: Use when accepting or returning a money amount, percentage, ISO date string, email address, URL, phone number, postal code, temperature, geographic coordinate, or any data whose validity is more constrained than its built-in type. Use when typing a field as `string` or `number` even though only certain values are legal. Use when an `assert(amount >= 0)` appears in three different functions. Use when a function comment says "must be a valid ISO-8601 date" or similar.
---

# Value objects for domain primitives

## Overview

**Don't use built-in primitives (`string`, `number`, `Date`) for domain concepts whose values are constrained.** Wrap them in small types — value objects — where the constructor or parser is the *only* place the constraint is checked. Once you hold the type, you trust the value; no defensive re-checking anywhere else.

A `Money` is not a `number`. An `EmailAddress` is not a `string`. A `Percentage` is not a `number`. Each has invariants — non-negative, well-formed, in range — and a fixed unit. Wrapping them once eliminates an entire class of repeated guards.

## The default rule

```
Prefer a value object over a raw `string`/`number` for a constrained domain
concept. Validate once at construction; trust the value thereafter. Mandatory
once the constraint is enforced in more than one place — see below.
```

**When this doesn't apply:**
- **Mandatory once the same constraint is checked in ≥2 places, or the unit/scale is ambiguous.** Three functions each re-checking `amount >= 0`, or a `number` that's cents in one place and dollars in another, is exactly what the wrapper eliminates. The "When NOT to wrap" section of this skill draws the same line.
- **A one-off, locally-checked constrained value** — validated at the single point it's produced and consumed, never crossing a boundary — doesn't need its own type. Wrap by default; skip when the constraint genuinely lives in one place.

## Why

**Primitive obsession** — Fowler's name for it in *Refactoring* — is one of the most common smells. The pattern: a concept that *is* a thing (money amount, percentage, lat/lon) is represented as a raw `number` or `string`, with validation rules duplicated at every call site.

The cost:

1. **Repeated guards.** Every function that takes a `price: number` re-checks `if (price < 0)`. Three sites = three places to keep the rule in sync; they drift.
2. **Lost units / scale.** Is `amount: number` in cents or dollars? In USD or EUR? You read the calling context to find out. Bugs ship when a developer reads it wrong.
3. **No domain operations.** A `Money` should support `add`, `subtract`, `scaleBy`. With a raw `number`, those are inline arithmetic — including the bugs (floating-point cents, mixed currencies summed silently).
4. **No type-level distinction.** A function that takes `(price: number, discount: number)` accepts the arguments swapped. The type system shrugs.

The fix isn't *more* validation everywhere — it's *one* validation, at construction, and a type that propagates the proof.

## Detection

You are violating the rule if any of these are true:

- A function signature has `amount: number`, `price: number`, `percentage: number` without a value-object wrapper.
- A field is typed as `string` with a code comment "must be a valid email" or "ISO-8601 date" or similar.
- The same validation regex / range check appears in 3+ files.
- Worse than a re-checked range: re-implemented *conversion* logic. A `convertToSmallestUnit(amount, currency)` copy-pasted across modules carries an embedded constant table (which currencies are zero-decimal) that drifts between copies — so the same input converts differently depending on which file you import. That's a correctness bug, not just duplication; one `Money` factory owns one table.
- A money amount lives in a binary float — a fractional-dollar `number`, a `real`/`double precision` column — instead of integer minor units or a decimal type.
- A bug post-mortem cites "passed cents but expected dollars" or "wrong currency added."
- Two function parameters of the same primitive type can be swapped without a typecheck error, and getting the order wrong is a real risk.

## The pattern

### Money

```ts
// ❌ Raw number. Unit ambiguity, no operations, every site re-validates.
function chargeCustomer(customerId: CustomerId, amount: number) {
  if (amount < 0) throw new Error('negative amount');
  if (amount > 100_000_000) throw new Error('too large');
  /* ... */
}

function sumLineItems(items: { price: number }[]): number {
  return items.reduce((sum, it) => sum + it.price, 0);
}
```

```ts
// ✅ Value object. Unit, invariants, operations — and currency scale — centralized.
type Currency = 'USD' | 'EUR' | 'JPY'; // a closed set you control; JPY is zero-decimal

export class Money {
  private constructor(
    readonly minorUnits: number, // smallest unit for THIS currency (cents, yen…)
    readonly currency: Currency,
  ) {}

  static of(minorUnits: number, currency: Currency): Money {
    if (!Number.isSafeInteger(minorUnits)) {
      throw new RangeError(`Money must be integer minor units, got ${minorUnits}`);
    }
    if (minorUnits < 0) {
      throw new RangeError(`Money cannot be negative, got ${minorUnits}`);
    }
    return new Money(minorUnits, currency);
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new Error(`Cannot add ${this.currency} to ${other.currency}`);
    }
    // Route through `of`, never `new Money` — so the sum is re-checked for overflow
    // past MAX_SAFE_INTEGER and any other invariant, instead of silently bypassing them.
    return Money.of(this.minorUnits + other.minorUnits, this.currency);
  }

  toString(): string {
    // Derive minor→major scale from the platform's ISO 4217 data — never a
    // hand-maintained decimals table (that's a stale duplicate, the very
    // primitive-obsession this object exists to kill). JPY → 0 digits, USD → 2.
    const fmt = new Intl.NumberFormat('en-US', { style: 'currency', currency: this.currency });
    const digits = fmt.resolvedOptions().maximumFractionDigits ?? 2;
    return fmt.format(this.minorUnits / 10 ** digits);
  }
}

// Consumers no longer guard:
function chargeCustomer(customerId: CustomerId, amount: Money) {
  // `amount` is already valid. Just use it.
}
```

The integer `minorUnits` field is itself the load-bearing choice, not a style preference. IEEE-754 binary floats cannot represent most decimal fractions — `0.1 + 0.2 !== 0.3` — so float money accumulates off-by-a-cent errors that surface months later as reconciliation failures nobody can trace to a single line. Store integer minor units (or a true decimal type — Postgres `numeric`, never `real`/`double precision`) end to end, and convert to display units only at the formatting edge, as `toString` does above.

Whether money may be **negative** is a per-domain decision, not a universal law: a charge amount is non-negative, but a refund, a ledger delta, or an account balance is legitimately signed. The example above enforces non-negative as *one* domain's rule — model a signed `LedgerAmount` separately rather than forcing every money type through the same non-negativity check.

The class form is one of two valid shapes. The other is a branded type — pick based on whether you need methods:

```ts
// Alternative: branded type for value-only cases.
import type { Brand } from './brand';
export type NonNegativeMinorUnits = Brand<number, 'NonNegativeMinorUnits'>;

export const NonNegativeMinorUnits = {
  of(n: number): NonNegativeMinorUnits {
    if (!Number.isInteger(n) || n < 0) throw new RangeError(`invalid: ${n}`);
    return n as NonNegativeMinorUnits;
  },
};
```

Use the **class** when you need behavior (`.add()`, `.subtract()`, `.toString()`). Use the **brand** when you only need the type-level distinction.

### Percentage

```ts
// ❌ Range, scale, and "is this a fraction or a percent?" all ambiguous.
function applyDiscount(price: number, discount: number): number {
  return price * (1 - discount);
}
applyDiscount(100, 0.1);  // 90 — discount as fraction
applyDiscount(100, 10);   // -900 — caller passed percent, oops
```

```ts
// ✅ Value object encodes the unit + range.
export type Percentage = Brand<number, 'Percentage'>; // stored as 0..1 fraction

export const Percentage = {
  fromFraction(f: number): Percentage {
    // `!Number.isFinite` first: `NaN < 0 || NaN > 1` is `false`, so NaN would slip past a bare range check.
    if (!Number.isFinite(f) || f < 0 || f > 1) throw new RangeError(`Percentage out of range: ${f}`);
    return f as Percentage;
  },
  fromPercent(p: number): Percentage {
    if (!Number.isFinite(p) || p < 0 || p > 100) throw new RangeError(`Percent out of range: ${p}`);
    return (p / 100) as Percentage;
  },
};

function applyDiscount(price: Money, discount: Percentage): Money {
  return Money.of(Math.round(price.minorUnits * (1 - discount)), price.currency);
}

// Construction is explicit about unit:
applyDiscount(price, Percentage.fromPercent(10));    // ✅
applyDiscount(price, Percentage.fromFraction(0.1));  // ✅
applyDiscount(price, 10);                            // ❌ compile error
```

### Email

```ts
// ❌ Every function that takes an email re-validates.
function sendEmail(to: string, subject: string) {
  if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(to)) throw new Error('invalid email');
  /* ... */
}

// ✅ Parse once at the boundary.
export type EmailAddress = Brand<string, 'EmailAddress'>;

const emailSchema = z.email();

export const EmailAddress = {
  parse(raw: unknown): EmailAddress {
    return emailSchema.parse(raw) as EmailAddress;
  },
};

// Internal consumers trust:
function sendEmail(to: EmailAddress, subject: string) {
  // `to` is already validated.
}

// Boundary calls parse:
export async function handleContactSubmit(body: unknown) {
  const input = ContactInput.parse(body);
  const to = EmailAddress.parse(input.to);
  await sendEmail(to, input.subject);
}
```

### ISO-8601 date

```ts
// ❌ Date format implicit.
function logEvent(at: string, name: string) {
  // assume `at` is ISO-8601 — fingers crossed
}

// ✅ Date format encoded in the type.
import { Temporal } from '@js-temporal/polyfill'; // use native Temporal where available

export type IsoDate = Brand<string, 'IsoDate'>;

export const IsoDate = {
  of(d: Date): IsoDate {
    return d.toISOString() as IsoDate;
  },
  parse(s: string): IsoDate {
    // Shape check…
    if (!/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(\.\d+)?(Z|[+-]\d{2}:\d{2})$/.test(s)) {
      throw new Error(`Not an ISO date: ${s}`);
    }
    // …then strict calendar validity. `Date.parse` is NOT enough: it normalizes
    // `2024-02-30T00:00:00Z` to March 1 instead of rejecting it.
    try {
      Temporal.Instant.from(s);
    } catch {
      throw new Error(`Impossible date: ${s}`);
    }
    return s as IsoDate;
  },
};
```

### Lat/Lon

```ts
// ❌ Two raw numbers; trivial to swap.
function distance(lat1: number, lon1: number, lat2: number, lon2: number) { /* ... */ }
distance(coord.lon, coord.lat, other.lat, other.lon); // ouch

// ✅ Single named coordinate type; constructor checks ranges.
export type Latitude = Brand<number, 'Latitude'>;
export type Longitude = Brand<number, 'Longitude'>;
export type Coordinates = { lat: Latitude; lon: Longitude };

export const Latitude = {
  of(n: number): Latitude {
    // `!Number.isFinite` first — otherwise NaN passes the range check (both comparisons are false).
    if (!Number.isFinite(n) || n < -90 || n > 90) throw new RangeError(`lat out of range: ${n}`);
    return n as Latitude;
  },
};
// ... similar for Longitude

function distance(from: Coordinates, to: Coordinates) { /* ... */ }
```

The function signature now reads naturally and can't be called with arguments swapped.

### When NOT to wrap

Some primitives are fine raw:

- **Loop counters, array indices** — `for (let i = 0; i < arr.length; i++)`.
- **Local-only intermediate values** with no domain meaning — `const half = total / 2`.
- **Truly unconstrained text** — a `description: string` where any string is valid.

The rule applies to **domain primitives** — values where the type system could carry more information than `string` or `number` does. Loop indices aren't domain primitives.

## Pressure resistance

### "It's just a number — wrapping is overkill"

It's not just a number. A `price` has a unit (cents/dollars), a sign (non-negative), a maximum, and a currency. Each is a constraint. Each is checked somewhere. The wrap centralizes them once. Net code is *less*, not more.

### "I'd have to wrap and unwrap everywhere"

You wrap at the *boundary* (input parsing) and unwrap at the *boundary* (output serialization). The middle (90% of the code) holds value-objects and never touches the raw primitive. The wrap-unwrap surface is small and intentional. (In codebases with many shared DTO/ORM-input interfaces, expect that boundary to be wider than two functions — you convert at each repository/serialization edge. The interfaces stay raw; the domain layer between them holds value objects.)

### "We already pass amount and currency together as one object"

Grouping two raw fields in a bag (`{ amount: number; currency: string }`) gives them no constructor, no invariant, and no distinct type. Any other `{ amount, currency }` bag is assignable to it, the amount can still be the wrong unit, and every consumer still re-converts. A value object is the bag *plus* the enforced construction — `Money.of(minorUnits, currency)` — so an invalid instance can't exist and the scale lives in one place.

### "TypeScript's type alias does the same thing"

A type alias (`type Email = string`) is structurally `string` — the compiler treats them identically. A branded type or class is structurally distinct. The whole point is that `EmailAddress` is *not* assignable from a raw `string` without going through `.parse()`.

### "Performance — class allocations / brand casts"

Branded types are zero-runtime: the brand is a phantom. Classes do allocate, but the cost is negligible compared to anything else the program does (DB round-trips, JSON serialization). If profiling shows allocations are hot, switch that one site to branded form. Don't pre-optimize.

### "We use schema parsing at the boundary, that's enough"

Schemas parse at the boundary — good. But the *output* of `z.email().parse(input)` is still typed as `string`. Internal functions accepting that value re-receive an unbranded `string` and can be called with any other string. The value object goes one step further: the parsed result has its own type that propagates.

### "Different code style — we're functional"

Branded types are functional. You don't need classes; you need types and a parsing function. The branded-type form (`Percentage`, `EmailAddress`, `IsoDate`) is functional and FP-friendly.

## Red flags

- Two parameters of the same primitive type whose order matters semantically.
- The phrase "must be a valid X" in a JSDoc comment, code comment, or type annotation.
- A regex literal or range-check that appears in more than one file for the same domain concept.
- A `Math.round` / `parseFloat` / `Number(...)` in three different places for the same domain value.
- A function takes `price` and `taxRate` and silently produces wrong results when the caller passes them in the wrong unit or scale.
- The bug post-mortem mentions "wrong unit", "wrong currency", "swapped arguments", "off by 100x."
- A test exercises a function with values that violate the function's documented preconditions but typecheck.

**All of these mean: the value-object is missing — wrap the constraint into a type.**

## Common rationalizations

| Excuse | Reality |
|---|---|
| "It's a value, not an entity" | Value objects are exactly *for* values. The class/brand is the value-object form. |
| "We don't need methods, just data" | Then use the branded-type form. No methods, zero runtime cost, still type-distinct. |
| "It's only used in one place" | Then it's free to add the wrap — and the next consumer who appears doesn't re-validate. |
| "Wrapping is an extra layer" | The layer is the validation. The "extra layer" was already there, just scattered across call sites. |
| "DB rows have raw numbers" | Convert at the DB boundary (a small parse-on-read helper). Internally hold value-objects. |
| "Wrapping makes the JSON serialization weird" | A `toJSON()` method on the class (or a thin serialization layer for brands) handles it cleanly. |

## Related

- `branded-ids` — the same wrapping technique, for IDs rather than constrained values
- `parse-dont-validate` — value objects are what parsing produces at the boundary

## Reference

- Eric Evans, *Domain-Driven Design* (2003) — coined the term **value object** as a first-class building block of domain models: "An object that represents a descriptive aspect of the domain with no conceptual identity is called a VALUE OBJECT."
- Martin Fowler, *Refactoring* 2e (2018) — names the smell as **primitive obsession** and lists refactorings that target it (Replace Primitive with Object, Extract Class).
- Vaughn Vernon, *Implementing Domain-Driven Design* (2013) — extends Evans' framing with concrete patterns that translate cleanly into TS.
