---
name: open-closed
description: Use when adding a new variant to an existing union, a new case to a switch, a new handler to a router, a new payment method, a new file format, or any other "new kind of X." Use when reviewing a PR that edits an `if/else` branch to add a new condition rather than adding a new variant. Use when tempted to add a `type` parameter to handle a new case via flag-driven branching.
---

# Open/Closed Principle

## Overview

**Modules should be open for extension and closed for modification.** Add new behavior by *adding new code* — a new variant, a new module, a new handler — not by editing existing branching. The existing branches stay stable; the new behavior lands as new code that the type system or registry connects.

In a TS/functional stack, this most often translates to: **add a new variant to a discriminated union and handle it in every consumer (the compiler enforces this via exhaustiveness checks) rather than adding a new `if` branch inside an existing function.**

## The Iron Rule

```
NEVER add a new kind by editing an existing if-chain. Add a variant; let the compiler force consumer updates.
```

**No exceptions:**
- Not for "adding an `if` is faster"
- Not for "we don't know what variants will exist"
- Not for "I just need to add one if"
- Not for "discriminated unions are TS-specific magic"

## Why

The classical OO statement (Meyer 1988; Martin 1996) is about class inheritance — you should extend a class by subclassing rather than editing the base. That framing doesn't fit a functional TS codebase. But the underlying idea translates cleanly and remains valuable.

**The underlying idea:** when a module has been verified to work — by code review, by tests, by production runtime — every edit to it is a *re-verification* cost. New branching inside a stable module risks regressions in existing branches. Extension that *adds* new code (with exhaustiveness checks ensuring every consumer handles the new case) is structurally safer.

The wins:

1. **Existing tests stay green.** Pure additions don't break existing assertions.
2. **The compiler enforces consumer updates.** Adding a variant to a union forces every `switch (kind)` to handle the new case — a moving safety net.
3. **Git history stays focused.** New behavior = new files / new variants. The diff localizes to the change.
4. **Reasoning stays bounded.** "What does this module do?" — read the registry/union once, understand the closed set.

The cost: a small amount of up-front design (you have to identify the right extension point — usually a discriminated union or a registry).

## Detection

You are violating the rule if any of these are true:

- A function gains a new `if (type === 'newKind')` branch every time the team adds a new kind.
- A switch statement has 8+ cases and grows whenever the domain grows.
- A function signature gets a new optional parameter every time a new use case appears.
- A constructor or factory function takes a `type: string` and branches on it internally.
- A "Manager" or "Service" class accretes methods every time the domain adds an entity.
- A configuration object grows a new field for every new behavior toggle.
- The phrase "add another flag to the function" appears in code review.

## The Pattern

### Discriminated union as the extension point

```ts
// ❌ A single function with a growing switch. Each new event type edits this function.
function handleEvent(event: { type: string; data: unknown }) {
  if (event.type === 'invoice.created') {
    return chargeCustomer(event.data);
  }
  if (event.type === 'invoice.paid') {
    return markPaid(event.data);
  }
  if (event.type === 'invoice.refunded') { // ← editing the function to add this
    return refund(event.data);
  }
  throw new Error(`Unknown event: ${event.type}`);
}
```

```ts
// ✅ Discriminated union + exhaustive switch. New variants are *additions* —
//    the compiler forces every consumer to handle them.
type InvoiceEvent =
  | { kind: 'created';  invoice: Invoice }
  | { kind: 'paid';     invoiceId: InvoiceId; paidAt: Date }
  | { kind: 'refunded'; invoiceId: InvoiceId; reason: string }; // ← added

function handleEvent(event: InvoiceEvent) {
  switch (event.kind) {
    case 'created':  return chargeCustomer(event.invoice);
    case 'paid':     return markPaid(event.invoiceId, event.paidAt);
    case 'refunded': return refund(event.invoiceId, event.reason); // ← compiler-required
    default:         return assertNever(event);
  }
}
```

The difference is subtle but important: in the second form, **adding `refunded` is a change to the *type* and a change to the *consumer*, in lockstep**. The compiler refuses to ship one without the other. In the first form, adding a new event type meant editing the function body — and if you forgot to update a *second* consumer of the same event type elsewhere in the codebase, the bug shipped.

### Registry / handler-map as the extension point

For larger systems where the variants live in different modules:

```ts
// ❌ A monolithic file owns all handlers and grows forever.
function dispatchPaymentMethod(method: string, amount: Money) {
  if (method === 'card')      return chargeCard(amount);
  if (method === 'bank')      return chargeBank(amount);
  if (method === 'apple_pay') return chargeApplePay(amount); // ← editing
  // ... 12 more
}
```

```ts
// ✅ Each payment method is a separate module. The registry is a small map.
//    Adding a new method = adding a new module + one entry in the registry.

// src/payments/card.ts
export const cardMethod: PaymentMethod = {
  kind: 'card',
  charge: (amount: Money) => { /* ... */ },
};

// src/payments/bank.ts
export const bankMethod: PaymentMethod = { kind: 'bank', charge: /* ... */ };

// src/payments/index.ts
import { cardMethod } from './card';
import { bankMethod } from './bank';
export const paymentMethods: Record<PaymentKind, PaymentMethod> = {
  card: cardMethod,
  bank: bankMethod,
};

// Consumer
export async function chargeWith(kind: PaymentKind, amount: Money) {
  return paymentMethods[kind].charge(amount);
}
```

`PaymentKind` is a union literal; the consumer reads through the map, not through branching. Adding `apple_pay`:

1. Add `'apple_pay'` to the `PaymentKind` union.
2. Create `src/payments/applePay.ts`.
3. Add one entry to the registry.

The compiler enforces that the registry has an entry for every union member (via `Record<PaymentKind, PaymentMethod>`). The consumer doesn't change.

### Strategy via function passing

For one-off variation points:

```ts
// ❌ A sort function with a flag-driven order parameter.
function sortInvoices(invoices: Invoice[], order: 'amount' | 'date' | 'customer') {
  if (order === 'amount')   return invoices.toSorted((a, b) => a.amount - b.amount);
  if (order === 'date')     return invoices.toSorted(/* ... */);
  if (order === 'customer') return invoices.toSorted(/* ... */);
}
```

```ts
// ✅ The sort criterion is a function — caller composes its own.
function sortInvoices(
  invoices: Invoice[],
  compare: (a: Invoice, b: Invoice) => number,
): Invoice[] {
  return invoices.toSorted(compare);
}

// Adding a new sort order is a new comparator at the call site —
// no change to sortInvoices.
sortInvoices(invoices, (a, b) => a.amount - b.amount);
sortInvoices(invoices, (a, b) => a.customer.localeCompare(b.customer));
```

This is OCP via higher-order functions — the standard TS-idiomatic form.

### When NOT to OCP

The rule applies when **the variants are likely to grow**. If you have a function that takes a `priority: 'low' | 'high'` and you're confident those are the only two values forever, an `if/else` is fine. Speculative OCP-ification — adding plugin points "in case we need them later" — is a YAGNI violation.

The signal that OCP is worth it: you've already added the *second* variant. Two is data; one is a guess.

## Pressure Resistance

### "Adding an `if` is faster than restructuring"

Faster the first time. Slower the third time, when the function has 8 branches and adding the 9th means re-verifying all 8. OCP pays off on the third+ variant; for two variants it's break-even.

### "We don't know what variants will exist"

That's an argument for *not* OCP-ing yet — wait for the second variant. But once it's there, the third is highly likely. Adopt the OCP form on variant #2, not #1.

### "Discriminated unions are TS-specific magic"

They're the literal manifestation of a type-theoretic sum type, present in every modern type system (TS, Rust, OCaml, Haskell, Kotlin). In TS specifically they're cheap, well-supported, and exhaustively checked.

### "The registry/map adds a layer"

The map is one indirection. The "non-OCP" version has the branching scattered — to grep for "where is `card` payment handled" you scan the entire monolithic function. With the registry, you look up `paymentMethods.card`. Less indirection, more localized reading.

### "Inheritance would be cleaner"

Inheritance was the canonical OCP mechanism in the OO era. In a functional TS stack, discriminated unions + higher-order functions are cleaner — no hidden vtable, no liskov-substitution concerns, no `super` chaining.

### "I just need to add one if"

If the next variant arrives in a week, this rationalization compounds. The check: have you added an `if` to this same function before? If yes, this is the third+ time — restructure.

## Red Flags

- A switch or if-chain that grows by one branch per quarter.
- A function whose body has the shape `if (type === 'A') ... else if (type === 'B') ... else if (type === 'C')`.
- A "Manager" or "Service" class that gains a method every time the domain adds an entity.
- A configuration object with 15+ boolean flags that interact.
- A code review comment "add another case for the new event."
- A git blame showing the same file edited by 5 different PRs, each adding one branch.
- A `default: throw new Error('unhandled type')` that catches new types at runtime (should fail at compile time via `assertNever`).

**All of these mean: the module is closed for the wrong axis — restructure so new behavior is an addition, not an edit.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "More variants is unlikely" | If you've added two, three is likely. Plan for the structure on variant two. |
| "It's a small function" | Small functions become large when each new feature adds one branch. The OCP shape keeps small things small. |
| "Inheritance/polymorphism is overengineered" | Agreed for inheritance — but the TS form (discriminated union + exhaustive switch) is *less* code than the if-chain it replaces. |
| "It's fine for now" | The "it's fine for now" rationalization compounds. The third variant is the breaking point. |
| "OCP is OO theory" | The principle (extension by addition, not modification) applies to any modular system. The mechanism varies. |
| "I'll restructure later" | The longer you wait, the more callers need restructuring too. Restructure early; it's cheaper. |

## Reference

- Bertrand Meyer, *Object-Oriented Software Construction* (1988) — coined the principle in its original (OO inheritance) form.
- Robert C. Martin, "The Open-Closed Principle" (1996) and later *Clean Architecture* (2017) — generalized the principle: depend on abstractions; extend by adding new implementations, not by modifying existing ones.
- Programming-language theory: sum types (discriminated unions) are the type-theoretic mechanism that makes the OCP form trivial in TS, Rust, OCaml, Haskell, and Kotlin.
