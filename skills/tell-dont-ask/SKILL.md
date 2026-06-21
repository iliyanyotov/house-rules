---
name: tell-dont-ask
description: Use when calling code reads an object's field, branches on it, then mutates the same or related field. Use when "feature envy" appears in code review — one module repeatedly accessing another's internals to make decisions. Use when validation logic lives outside the entity it validates. Use when a `switch` on `entity.status` appears in code that's *not* the entity's definition. Use when entities are plain DTOs / schema-inferred types with no methods and the same `entity.status` rule is duplicated across the services that handle them.
---

# Tell, Don't Ask

## Overview

**Don't query an object for its state and then decide what to do based on that state from the outside.** Instead, *tell* the object what you want to happen — pass the intent in, and let the module that owns the data decide how to fulfill it.

The shift is from **caller-driven logic** ("I read your state and decide") to **module-driven logic** ("I tell your module my intent; it handles the rest").

## The Iron Rule

```
NEVER read another module's state to make a decision about that module's data. Tell the module what you want.
```

**No exceptions:**
- Not for "it's just one if-check"
- Not for "the module would have too many methods"
- Not for "it's not OO — we don't have objects"
- Not for "sometimes I really do need to read the state"

## Why

When code outside a module reads its state and decides what to do, the *rules of that module* live in the caller. Multiple callers inevitably duplicate the rules — and they drift. The classic symptom: "we changed how `status === 'paid'` is computed and now three places do it differently."

When the module owns the operation, the rule has one home. Callers say *what they want* (mark paid, send receipt, refund); the module says *how it works*. The wins:

1. **No rule duplication.** The "what does paid mean?" logic lives once.
2. **No invariant leak.** The module can enforce "you can't refund what was never paid" without trusting callers to check first.
3. **Refactors localize.** Change the rule once; every consumer benefits without edits.
4. **Tests test the rule, not the consumer's interpretation of the rule.**

This is *not* an object-oriented thing — it's a *cohesion* thing. The rule applies to functional modules, free functions, and entities equally. The point is: **the module that owns the data also owns the operations on it**.

## Detection

You are violating the rule if any of these are true:

- A function in module A reads a field on a B-typed object, branches on it, and then mutates B or calls another function on B.
- The same `if (entity.status === ...)` check appears in 3+ files outside the entity's own module.
- A "feature envy" code smell — a function that uses another module's data more than its own (Fowler).
- A `switch (entity.kind)` appears outside the entity's own module, and each branch does different mutations to the entity.
- Validation rules live in a separate `validators/` directory disconnected from the types they validate.
- A workflow function reads an object's fields, makes a decision, then calls a setter on the same object — could be one function call.

## The Pattern

### The classic example — refund

```ts
// ❌ The caller knows the rule for "can refund."
// Three places duplicate this. Drift inevitable.
function refundIfPaid(invoice: Invoice) {
  if (invoice.status === 'paid' && invoice.paidAt && !invoice.refundedAt) {
    invoice.status = 'refunded';
    invoice.refundedAt = new Date();
    queueRefundPayment(invoice.amount, invoice.customerId);
  }
}
```

```ts
// ✅ The invoice's module owns the operation. Callers express intent.
// src/domain/invoice.ts
export function refundInvoice(invoice: Invoice, now: Date): RefundResult {
  if (invoice.status !== 'paid' || !invoice.paidAt) {
    return { ok: false, reason: 'not-paid' };
  }
  if (invoice.refundedAt) {
    return { ok: false, reason: 'already-refunded' };
  }
  return {
    ok: true,
    invoice: { ...invoice, status: 'refunded', refundedAt: now },
    refundAmount: invoice.amount,
  };
}

// Consumer
const result = refundInvoice(invoice, new Date());
if (result.ok) {
  await invoiceRepo.save(result.invoice);
  await paymentQueue.enqueueRefund(result.refundAmount, invoice.customerId);
}
```

The consumer doesn't know what "paid" means or what fields constitute "refunded" — it asks for the operation; the module enforces the rules. Two consumers calling `refundInvoice` get *the same* rules.

### Pass intent, don't query state to decide

```ts
// ❌ Querying state, deciding outside.
if (user.subscription.tier === 'free' && user.usage > FREE_LIMIT) {
  showUpgradePrompt(user);
}

// ✅ Ask the module if the user should be prompted.
if (shouldPromptUpgrade(user)) {
  showUpgradePrompt(user);
}
```

The "free tier exceeds limit" rule lives once, in `shouldPromptUpgrade`. Adding a "trial tier" or a "grandfathered users skip the prompt" exception is a one-place change.

### Compose tells, not asks

```ts
// ❌ A pipeline that reads, decides, reads, decides…
async function processOrder(orderId: OrderId) {
  const order = await orderRepo.findById(orderId);
  if (order.status === 'pending') {
    if (order.paymentMethod === 'card') {
      const result = await chargeCard(order.amount);
      if (result.ok) {
        order.status = 'confirmed';
        order.confirmedAt = new Date();
        await orderRepo.save(order);
      } else {
        order.status = 'failed';
        order.failureReason = result.error;
        await orderRepo.save(order);
      }
    }
    // ... more branches
  }
}
```

```ts
// ✅ A pipeline that tells each step what to do.
async function processOrder(orderId: OrderId) {
  const order = await orderRepo.findById(orderId);
  const result = await attemptConfirmation(order, paymentProcessor);
  await orderRepo.save(result.order);
  if (result.kind === 'failed') {
    await notifyCustomerOfFailure(order.customerId, result.reason);
  }
}

// src/domain/order.ts — owns the rule
async function attemptConfirmation(
  order: Order,
  paymentProcessor: PaymentProcessor,
): Promise<ConfirmationResult> {
  if (order.status !== 'pending') {
    return { kind: 'skipped', order, reason: 'not-pending' };
  }
  const charge = await paymentProcessor.charge(order.amount, order.paymentMethod);
  if (!charge.ok) {
    return {
      kind: 'failed',
      order: { ...order, status: 'failed', failureReason: charge.error },
      reason: charge.error,
    };
  }
  return {
    kind: 'confirmed',
    order: { ...order, status: 'confirmed', confirmedAt: new Date() },
  };
}
```

The consumer reads top-to-bottom: find, attempt, save, notify-if-failed. The decisions live in `attemptConfirmation`.

### Validation belongs with the type

```ts
// ❌ Validators in a separate "validation" layer.
// src/validators/emailValidator.ts
export function validateEmail(s: string): boolean { /* ... */ }

// Consumer
if (validateEmail(input.email)) {
  user.email = input.email;
}
```

```ts
// ✅ The type owns its validation.
// src/domain/email.ts
export const EmailAddress = {
  parse(raw: unknown): EmailAddress {
    if (typeof raw !== 'string' || !EMAIL_REGEX.test(raw)) {
      throw new InvalidEmailError(raw);
    }
    return raw as EmailAddress;
  },
};

// Consumer
const email = EmailAddress.parse(input.email);
user.email = email;
```

The boundary is sharp: outside the email module, you have `unknown`; inside, you have `EmailAddress`. The transition (the parse) *is* the validation; there's no separate "validator" abstraction.

### When to ask (the exceptions)

Tell-Don't-Ask isn't absolute. **Ask is fine when**:

- The query is purely informational (rendering UI, logging, debugging).
- The caller's *only* purpose is to display the data (a view layer reads model fields freely).
- The decision the caller makes is genuinely the caller's responsibility, not the queried module's.

The line: **does the decision encode a rule of the queried module?** If yes, the rule should live there. If no — if the caller is just adapting data for its own purpose (rendering, formatting, aggregating across modules) — asking is fine.

```ts
// ✅ Asking is fine here — the UI is rendering, not deciding business rules.
<p>Status: {invoice.status}</p>
{invoice.paidAt && <p>Paid: {invoice.paidAt.toISOString()}</p>}

// ❌ Asking is not fine here — the UI is making a business decision.
{invoice.status === 'paid' && !invoice.refundedAt && (
  <RefundButton onClick={() => refundIt(invoice)} />
)}
```

The second case: `canBeRefunded(invoice)` belongs in the invoice module.

### When entities are plain data (DTOs across a boundary)

A common architecture makes entities **plain serializable types with no methods** — schema-inferred shapes shared between a server and a client, or DTOs that cross a network/process boundary. You *cannot* put `invoice.refund()` on such a type; it must stay a bare data structure to serialize cleanly. It's tempting to conclude "Tell-Don't-Ask needs rich objects, so it doesn't apply here." That conclusion is wrong, and it's the source of a false-positive reading of this skill.

The principle was never about *objects*; it's about **which module owns the decision** (see Why). The unit that owns the rule isn't a method on the entity — it's a **dedicated domain module for that entity**: one file (`domain/invoice.ts`) exporting free functions that take the entity and return a result or a new entity. Every solution example in this skill is already in that form — `refundInvoice(invoice, now)`, `attemptConfirmation(order, …)`, `shouldPromptUpgrade(user)`. None is a method; all are "tell."

So in a DTO codebase the rule sharpens rather than disappears:

```ts
// ❌ The "ask" smell, DTO edition: every service that touches a booking
//    re-implements the cancellation rule by reading the DTO's fields.
// src/services/booking/cancelBookingService.ts
if (booking.status === 'confirmed' && booking.startsAt > now && !booking.cancelledAt) {
  // ... cancel
}
// src/services/booking/rescheduleBookingService.ts  — the SAME rule, drifting
if (booking.status === 'confirmed' && booking.startsAt > now) { /* ... */ }

// ✅ One domain module owns the rule; services tell.
// src/domain/booking.ts  (operates on the plain DTO, returns a new DTO)
export function canCancel(booking: Booking, now: Date): boolean {
  return booking.status === 'confirmed' && booking.startsAt > now && !booking.cancelledAt;
}
export function cancelBooking(booking: Booking, now: Date): CancelResult {
  if (!canCancel(booking, now)) return { ok: false, reason: 'not-cancellable' };
  return { ok: true, booking: { ...booking, status: 'cancelled', cancelledAt: now } };
}
```

The entity stays a pure DTO; the *behavior* concentrates in one named module per entity instead of smearing across every service that handles it. If your services each carry their own `if (entity.status === …)` rule, that's the violation — not the absence of methods on the entity.

**The guard:** "entities must be plain data" justifies *no methods on the type*. It does **not** justify scattering the entity's rules across its callers. Give the entity a domain-function module and tell *it*.

## Pressure Resistance

### "It's just one if-check"

The first one is. The second copy is in another file. The third copy is in the test. Now there are three places to change the rule. The pattern compounds quickly.

### "The module would have too many methods"

The module would have the *right* methods — one per intent. The alternative is the same methods spread across consumers, duplicated. Fewer modules with cohesive methods beat many consumers with scattered logic.

### "It's not OO — we don't have objects"

The "object" in "Tell-Don't-Ask" is a misleading historical label. The principle is about *which module owns which decision*, regardless of whether classes are involved. A free function `refundInvoice(invoice, ...)` is "tell." A consumer that reads `invoice.status` and decides what to do is "ask."

### "It hides the logic"

It *moves* the logic to where it belongs — the module that owns the data. From the consumer's view, the logic is one function call away with a clear name. That's hiding implementation, not hiding behavior.

### "Sometimes I really do need to read the state"

Sure — for *display*. The skill applies to *decisions*. If you're reading to render, ask freely. If you're reading to mutate or invoke side effects, tell.

### "Our entities are plain DTOs with no methods — this skill can't apply"

It applies, just not as methods. The owner of the rule is a per-entity *domain module* (`domain/invoice.ts`) of free functions that take the DTO and return a new one — exactly the form every example here uses. "No methods on the type" is fine; "the rule duplicated across every service that reads the DTO" is the violation. See *When entities are plain data*.

### "I can't refactor everything"

Apply incrementally. Next time you'd write `if (entity.x === ...)` outside the entity's module, write `if (entityShouldX(entity))` instead and put the rule in the module. Over months, the codebase converges.

## Red Flags

- A `switch (x.kind)` outside the file that defines `x.kind`'s union.
- A function that reads 3+ fields from a parameter and decides based on the combination.
- The same `if (entity.field === literal)` check in 3+ files.
- A `validators/` directory disconnected from the types it validates.
- A workflow function whose body is a long `if/else` over entity state.
- A bug post-mortem where "we forgot to check X in one place." The rule was duplicated; only one copy got fixed.
- A "feature envy" smell — function uses another module's fields more than its own arguments.

**All of these mean: the decision belongs in the module that owns the data, not in the caller.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The caller knows what to do" | Until two callers disagree on what to do, and the bug ships. |
| "Adding a method to the module is overkill" | Adding a function to the module is one line of organization vs. the rule's permanent duplication. |
| "It's just glue code" | Glue code that encodes a rule isn't glue — it's a load-bearing decision. |
| "I'd have to import the type just to call it" | Same import as the field access. No additional cost. |
| "It's pure logic — why hide it?" | The pure logic stays exposed (as a function in the module). What's hidden is the *decision tree* — which belongs in the module that owns the data. |
| "Tell-Don't-Ask is OOP dogma" | The principle is about cohesion, not classes. Functional modules benefit equally. |

## Reference

- Andy Hunt and Dave Thomas, *The Pragmatic Programmer* (1999, 20th-anniversary ed. 2019) — named the principle "Tell, Don't Ask." The underlying cohesion argument carries across functional and OO codebases.
- Martin Fowler, *Refactoring* (2nd ed., 2018) — the related code smells **Feature Envy** and **Inappropriate Intimacy** name the violations of this principle.
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (2009) — frames the same idea from a TDD perspective: design with messaging in mind, not with state-then-conditional in mind.
