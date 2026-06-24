---
name: errors-as-values
description: Use when a function can fail in more than one distinguishable way and the caller must react differently to each. Use when modeling an error channel as `string`, bare `Error`, or `null`. Use when a `try`/`catch` block inspects `err.message` or `err instanceof X` to decide what to do. Use when failures need to propagate through several layers without each one re-wrapping them. Use when a `catch` swallows an error and the caller can no longer tell what went wrong.
---

# Errors as Values

## Overview

**When a failure is real and the caller must distinguish its causes, the default is to model it as a typed value — a discriminated `Result<T, E>` whose `E` is an exhaustive union — not a `string`, not a bare `Error`.** The success and the failure both travel in the return type. The caller pattern-matches; the compiler enforces that every failure case is handled. (There is one disciplined alternative — a typed error thrown to a *single* top-level boundary; see "When throwing is the right call." What's never acceptable is a *stringly-typed* error, return or throw.)

A thrown exception is invisible in the signature: nothing tells the caller it can happen, what causes it, or which causes are distinct. A typed error value is the opposite — it is *in the type*, exhaustive, and checkable.

## The Iron Rule

```
NEVER signal a distinguishable, recoverable failure with a stringly-typed error. Return Result<T, E> with E a discriminated union by default — or, as a disciplined alternative, throw a *typed* error to one top-level boundary (see "When throwing is the right call"). Never a bare string.
```

**No exceptions:**
- Not for "the error message is enough"
- Not for "I'll just throw and catch it upstairs"
- Not for "there's only one failure case today"
- Not for "`catch (e)` and log is fine"

## Why

Three things go wrong when failures are thrown or stringly-typed.

**The signature lies.** `function charge(): Receipt` claims it returns a receipt. It actually returns a receipt *or* throws one of four things. The caller learns the failure modes by reading the body, getting paged, or guessing. A `function charge(): Result<Receipt, ChargeError>` states the contract.

**The caller can't branch safely.** When the error is a `string` or a bare `Error`, distinguishing "card declined" (show the user) from "provider timed out" (retry) means matching on `err.message` substrings or fragile `instanceof` chains. Both rot the first time someone rewords a message or wraps an error — and `instanceof` rots in another way too: a class duplicated across two bundled copies of a package fails its *own* `instanceof` check (different realm, different constructor identity), which is why production code falls back to a stable `code` string. A stable discriminant survives rewording *and* realm boundaries; `instanceof` survives neither reliably. A discriminated `E` lets the caller `switch` on `error.kind` with compile-time exhaustiveness.

**Propagation loses information.** A `try`/`catch` three layers up catches *everything* — the declined card, the timeout, and the genuine bug (a `TypeError` from a typo) all land in the same block, flattened to one `catch (e)`. Returning values keeps each failure named all the way up, and lets a real bug keep throwing.

The alternative to `try`/`catch` sprawl is not "swallow it." It is **make the failure a value with a typed shape, and let the type system carry it.**

## Where this sits among its neighbors

This rule is easy to confuse with three others. The boundaries:

- **define-errors-out-of-existence** asks: *can this failure not exist at all?* Clamp the range, return `T | undefined`, narrow the input type so the bad case is unconstructable. **Try that first.** `errors-as-values` is for the failures that *remain* — the ones that are genuinely possible and where the caller must tell the causes apart. If `T | undefined` says everything the caller needs (one failure mode, no detail required), use that — don't reach for `Result`.
- **fail-fast** is for *invariants you control* — a violated precondition, a missing env var, a `null` that your own code promised wouldn't be `null`. Those are bugs; **throw**. `errors-as-values` is for *expected* failures the caller is meant to handle.
- **make-illegal-states-unrepresentable** / **exhaustive-switch** are the *machinery* this rule runs on: `Result` is a discriminated union, and the caller handles it with an exhaustive switch. This skill is the application of those two to the error channel specifically.

One line: *make it disappear (define-errors) → if it's a bug, throw (fail-fast) → if it's a real, distinguishable failure the caller handles, return it as a typed value (this rule).*

## Detection

You are violating the rule if any of these are true:

- A `catch` block inspects `err.message` (substring match) or chains `instanceof` to decide what to do.
- A central error mapper that switches on a typed `code` for known errors but *falls back* to matching the raw `message` for everything else — the fallback re-opens the stringly-typed channel at the exact place meant to close it. The discipline is only as strong as its weakest branch.
- A function's error channel is typed `string`, `Error`, `unknown`, or `{ error: string }` when the caller branches on *which* error.
- A failure is thrown across two or more layers purely so something upstairs can `catch` it.
- A `Result<T, E>` exists but `E` is `string` — the union got flattened to a message.
- A caller of a `Result`-returning function does `if (!r.ok) throw new Error(r.error)` — converting a value back into an exception.
- A bug (`TypeError`, `undefined is not a function`) and an expected failure (declined card) are caught in the same `catch`.

## The Pattern

### The Result type — one definition, used everywhere

```ts
// A success carries a value; a failure carries a typed error. Nothing else.
export type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

export const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });
```

This is the same discriminated-union shape as `make-illegal-states-unrepresentable`. No library required. (If you want chaining combinators — `map`, `andThen` — `neverthrow` and `Effect` provide them; the hand-rolled union is the canonical form and what the rest of this skill uses.)

### The error is a union, never a string

```ts
// ❌ Stringly-typed: the caller can only re-display or substring-match.
async function charge(order: Order): Promise<Result<Receipt, string>> {
  // ... returns err('card declined') | err('provider timeout') | err('no payment method')
}
// Caller is stuck: if (r.error.includes('declined')) ... — fragile, unexhaustive.

// ✅ A discriminated union names each cause. The caller branches on `kind`.
type ChargeError =
  | { kind: 'card_declined'; declineCode: string }
  | { kind: 'no_payment_method' }
  | { kind: 'provider_unavailable'; retryAfterMs: number };

async function charge(order: Order): Promise<Result<Receipt, ChargeError>> {
  const method = await getPaymentMethod(order.customerId);
  if (!method) return err({ kind: 'no_payment_method' });

  const res = await provider.charge(method, order.totalCents);
  if (res.status === 'declined') {
    return err({ kind: 'card_declined', declineCode: res.code });
  }
  if (res.status === 'unavailable') {
    return err({ kind: 'provider_unavailable', retryAfterMs: res.retryAfterMs });
  }
  return ok({ receiptId: res.receiptId });
}
```

Each variant carries exactly the data its handler needs (`declineCode`, `retryAfterMs`) — no more.

### The caller handles it exhaustively — no try/catch

```ts
const assertNever = (x: never): never => { throw new Error(`unhandled: ${JSON.stringify(x)}`); };

const result = await charge(order);
if (result.ok) {
  return showReceipt(result.value);
}

// `result.error` is ChargeError; the switch is exhaustive by compiler check.
switch (result.error.kind) {
  case 'card_declined':
    return promptDifferentCard(result.error.declineCode);
  case 'no_payment_method':
    return promptAddPaymentMethod();
  case 'provider_unavailable':
    return scheduleRetry(result.error.retryAfterMs);
  default:
    return assertNever(result.error); // adding a 4th variant fails the build here
}
```

`assertNever` is the same exhaustiveness guard as the `exhaustive-switch` skill. Add a failure mode to `ChargeError` and every caller that forgot to handle it stops compiling — the opposite of a new `throw` that no caller knows about.

### Propagation — early-return on `!ok`, the shape `try`/`catch` wanted to be

```ts
// Each step can fail with its own error; the union widens as we go.
type CheckoutError =
  | { kind: 'cart_empty' }
  | { kind: 'out_of_stock'; sku: string }
  | ChargeError;

async function checkout(cartId: CartId): Promise<Result<Order, CheckoutError>> {
  const cart = await loadCart(cartId);
  if (cart.lineItems.length === 0) return err({ kind: 'cart_empty' });

  const reservation = await reserveStock(cart);
  if (!reservation.ok) return err(reservation.error); // forward the error channel; drop the foreign success type

  const order = await placeOrder(cart);
  const charged = await charge(order);
  if (!charged.ok) return err(charged.error); // ChargeError flows into CheckoutError

  return ok(order);
}
```

`if (!r.ok) return err(r.error)` is the value-level equivalent of letting an exception bubble — but the error stays named and a genuine bug in `placeOrder` still throws instead of being swallowed. Forward the error with `err(r.error)`, not bare `return r`: `r` is a `Result<Reservation, …>`, whose success branch carries a `Reservation`, not the `Order` this function must return — re-wrapping keeps the error channel and drops the foreign success type.

### The boundary still converts to/from exceptions

```ts
// At the OUTERMOST edge, translate the typed error into the transport.
// The framework wants an HTTP status or a thrown error boundary; give it one.
export async function POST(req: Request) {
  const result = await checkout(cartId);
  if (result.ok) return json({ order: result.value }, { status: 201 });

  switch (result.error.kind) {
    case 'cart_empty':
    case 'out_of_stock':
      return json({ error: result.error }, { status: 409 });
    case 'card_declined':
    case 'no_payment_method':
      return json({ error: result.error }, { status: 402 });
    case 'provider_unavailable':
      return json({ error: result.error }, { status: 503 });
    default:
      return assertNever(result.error);
  }
}

// Going the OTHER way: a throwing SDK becomes a Result at the boundary, once.
async function reserveStock(cart: Cart): Promise<Result<Reservation, CheckoutError>> {
  try {
    return ok(await inventorySdk.reserve(cart.lineItems));
  } catch (e) {
    if (e instanceof OutOfStockError) {
      return err({ kind: 'out_of_stock', sku: e.sku });
    }
    throw e; // a real bug, not an expected failure — let fail-fast handle it
  }
}
```

The `try`/`catch` lives in *one* adapter, translating a known exception into a typed value and **re-throwing anything it doesn't recognize**. The interior never catches.

### When throwing is the right call — do it typed, not stringly

`errors-as-values` is the default this skill teaches, but it is not the only disciplined option, and a mature codebase sometimes does the opposite *well*. There is a second coherent strategy: **a typed error-class hierarchy that is thrown, caught by one top-level boundary, and rendered as a structured response.** It is a legitimate sibling of `Result`, not the anti-pattern.

The two strategies oppose the *same* enemy — the bare `throw new Error("...")` / `catch (e) { e.message }` channel that callers can't reliably discriminate. They differ only in *where the branching happens*:

- **Return a `Result`** when a *caller* must branch on each failure locally and you want the compiler to force handling at the call site — checkout deciding whether to prompt for a new card, retry, or add a payment method. The decision lives next to the call.
- **Throw a typed error to a boundary** when, in a layered server, *most* failures should propagate untouched through every intermediate layer to a *single* place that maps them to HTTP. A repository deep in a request stack has nothing useful to do with "invoice not found" except let it travel to the one handler that turns it into a `404`. Threading `if (!r.ok) return r` through six layers that only forward the error buys nothing the throw doesn't.

When you choose the throwing style, the bar is: **typed base + stable machine-readable code + `cause`-chaining + exactly one catch boundary.** Never bare strings.

The common halfway-house is a class hierarchy whose *only* discriminant is the class name (or `this.name`): it reads as disciplined but still forces the boundary onto `instanceof` chains and name-string comparison. The stable `type` / `code` *field* is the non-negotiable part — not the class hierarchy. A flat error with a stable `code` beats a deep hierarchy with none.

```ts
// ✅ One base every domain error extends. `type` is a STABLE, machine-readable
// discriminant (a wire contract clients switch on) — never the human `message`.
// `status` carries the transport mapping; `context` carries safe-to-expose detail;
// `cause` chains the underlying failure for logs without leaking it to the client.
abstract class AppError extends Error {
  abstract readonly type: string;   // e.g. 'invoice/not_found' — stable, versioned
  abstract readonly status: number; // HTTP status this maps to
  readonly context: Record<string, string | number>;

  constructor(message: string, context: Record<string, string | number> = {}, cause?: unknown) {
    super(message, { cause });       // native cause-chaining
    this.name = new.target.name;
  }
}

class InvoiceNotFoundError extends AppError {
  readonly type = 'invoice/not_found';
  readonly status = 404;
  constructor(invoiceId: string, cause?: unknown) {
    super(`Invoice ${invoiceId} not found`, { invoiceId }, cause);
  }
}

class PaymentDeclinedError extends AppError {
  readonly type = 'payment/declined';
  readonly status = 402;
  constructor(declineCode: string, cause?: unknown) {
    super('Payment was declined', { declineCode }, cause);
  }
}
```

Interior layers throw and stay oblivious — no `try`/`catch`, no re-wrapping:

```ts
async function loadInvoice(invoiceId: string): Promise<Invoice> {
  const row = await db.invoices.find(invoiceId);
  if (!row) throw new InvoiceNotFoundError(invoiceId); // propagates untouched
  return row;
}
```

One boundary catches the base and renders an RFC 7807 problem+json error — a structured body keyed by the stable `type`, *not* by a substring of the message:

```ts
// The SINGLE catch boundary. Everything below it just throws AppError subclasses.
function toProblemResponse(e: unknown): Response {
  if (e instanceof AppError) {
    log.warn(e.type, { context: e.context, cause: e.cause }); // cause stays server-side
    return json(
      { type: e.type, title: e.message, status: e.status, ...e.context },
      { status: e.status, headers: { 'content-type': 'application/problem+json' } },
    );
  }
  log.error('unhandled', { cause: e });   // a genuine bug — fail-fast territory
  return json(
    { type: 'about:blank', title: 'Internal Server Error', status: 500 },
    { status: 500, headers: { 'content-type': 'application/problem+json' } },
  );
}
```

This is `instanceof AppError` at exactly *one* site against a sealed base — a completely different thing from the `instanceof` chains in Detection, which branch on many ad-hoc error classes scattered across interior callers. Here the interior never discriminates; the boundary does it once.

In a real app that boundary usually fans *in* several foreign taxonomies you don't own — a schema parser's error, an ORM's, a payment SDK's — so it won't be a single `instanceof YourBase`; it's a handful of type-guards mapping each foreign shape to a `code`. The discipline is that this fan-in lives at *exactly one site* and the interior never branches — not that everything extends one base.

The line both strategies refuse to cross is the stringly-typed one. `throw new Error('invoice not found')` answered by `if (e.message.includes('not found'))` is the failure mode — whether you return or throw, the discriminant must be a typed `kind`/`type`, never a message. Pick `Result` or typed-throw by where the decision lives; never pick bare strings.

## Pressure Resistance

### "The error message is enough — I'll just return a string"

A message is for a human reading a log. A `kind` is for code making a decision. The moment a caller needs to *do* different things for different failures, a string forces substring-matching, which breaks the first time someone rewords the message. Name the cause in the type.

### "I'll throw and catch it upstairs — same effect, less code"

Not the same effect. A `throw` is invisible in the signature, catches your genuine bugs in the same block, and gives upstream code no compiler signal that the case exists. `if (!r.ok) return r` is one line, keeps the error named, and lets real bugs keep propagating.

### "There's only one failure case today"

Then `T | undefined` or a single-variant union is enough today — and the union *grows* without breaking callers' types silently: add a variant and the exhaustive switch flags every unhandled site. A `throw` added later is silent. Model the channel as a union from the start when more than one cause is plausible.

### "Result types don't compose — it's `if (!r.ok) return r` everywhere"

That repetition *is* the composition, and it's the same shape `try`/`catch` has — minus the invisible control flow. If the early-returns genuinely sprawl, that's the signal to reach for `andThen`/`map` combinators (neverthrow, Effect), not to go back to throwing.

### "My function does I/O, it has to be able to throw"

Two failure modes, two treatments — the same split as `define-errors-out-of-existence`. Expected, distinguishable failures (declined, out-of-stock, not-found) → return a typed `Result`. Infrastructure collapse and bugs (socket closed, OOM, `TypeError`) → throw, and let `fail-fast` carry them to the global handler.

### "The error union will explode into `Result<T, A | B | C | D>`"

A wide error union is a signal, not a defect — it tells you exactly how many ways this operation fails, in the type, where you can see it. If it's genuinely too many, the function is doing too much; split it. A function that throws four things has the same four failure modes — it just hides them.

## Red Flags

- A `catch` block branching on `err.message.includes(...)` or a chain of `instanceof`.
- A `Result<T, E>` whose `E` is `string`, `Error`, or `unknown`.
- `if (!r.ok) throw new Error(r.error)` — a typed value converted back into an exception.
- A single `try`/`catch` wrapping a multi-step operation where the steps fail for unrelated reasons.
- A function whose JSDoc `@throws` lists conditions the caller is expected to handle (vs. true invariants).
- A `catch (e) {}` or `catch (e) { log(e) }` that returns a default, erasing which failure occurred.

**All of these mean: an expected, distinguishable failure is being thrown or stringly-typed — model it as `Result<T, E>` with a discriminated `E`.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Throwing is idiomatic in JS" | The idiom is mid-shift — `safeParse`, tRPC, React Query, neverthrow, Effect all return tagged results. Lead, don't follow. |
| "A string error is simpler" | Simpler to *produce*, impossible to *branch on* safely. The caller pays for it with substring matches. |
| "Exhaustive unions are overkill for errors" | They're the only thing that makes "did I handle every failure?" a compiler question instead of a production discovery. |
| "Stack traces are better than values" | Put the cause in the variant (or the `cause` chain at the boundary). A stack trace is noise when the failure is "card declined." |
| "Returning errors clutters every signature" | The clutter is the contract becoming visible. A throwing function has the same failure modes, hidden. |
| "I'd have to change every caller" | Callers that branch on failure were going to anyway — better the compiler checks the branches than scattered `try`/`catch` guesses at them. |

## The Bottom Line

**A recoverable, distinguishable failure belongs in the return type, not in a `throw`.**

Make it disappear if you can. Throw it if it's a bug. Otherwise return `Result<T, E>` with `E` a discriminated union, propagate with `if (!r.ok) return r`, and convert to exceptions only at the outermost boundary.

## Related

- `define-errors-out-of-existence` — handoff: what remains after "can this not exist?"
- `fail-fast` — distinguishes recoverable failures (return) from bugs (throw)
- `exhaustive-switch` — the machinery: discriminated error union + exhaustive handling
- `make-illegal-states-unrepresentable` — a Result is a discriminated union

## Reference

- Joe Duffy, ["The Error Model"](http://joeduffyblog.com/2016/02/07/the-error-model/) (2016) — the architectural case for separating recoverable *errors* (return them as values) from unrecoverable *bugs* (abandon/throw). The clearest treatment of the split this skill rests on.
- Scott Wlaschin, *Domain Modeling Made Functional* (2018), ch. 10 ("Working with Errors") — "Railway-Oriented Programming": modeling the error channel as a type and composing functions that can fail. The functional source for `Result`/`Either`.
- Rust's [`Result<T, E>`](https://doc.rust-lang.org/book/ch09-00-error-handling.html) and the `recoverable vs. unrecoverable` (panic) distinction — the most widely-deployed embodiment of exactly this rule.
