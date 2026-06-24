---
name: principle-of-least-astonishment
description: Use when naming a function, type, or module. Use when a function does more than its name suggests — sends an email, writes to the DB, or modifies global state behind a tame-sounding name. Use when a "getter" mutates state. Use when a "validator" returns a different type from the value it validated. Use when a developer reading your code has to read the body to know what it does. Use when a function's behavior surprised a reviewer.
---

# Principle of Least Astonishment

## Overview

**A function, type, or module should behave the way its name and signature lead a reader to expect.** Side effects, exceptions, and return values must match the name. The right time to surprise a reader is never.

If the implementation conflicts with the obvious reading of the name, fix the name — or fix the implementation.

## The Iron Rule

```
NEVER let a name promise one thing while the body does another. Fix the name or fix the body.
```

If you have to read the body to know what the function does, the name is wrong.

## The Pattern

The rule has three dimensions:

1. **Names match behavior.** `compute` shouldn't mutate. `find` shouldn't throw on not-found (use `findOrThrow` if it does). `save` shouldn't read.
2. **Signatures match conventions.** A function returning `Promise<T>` must actually await something. Parameter order must match the convention callers expect.
3. **Side effects are discoverable.** If a function logs, queues a job, mutates a parameter, or talks to the network — its name should say so.

### Names match behavior

```ts
// ❌ A `get` that's actually a `fetch` (network call).
function getUser(id: UserId): Promise<User> {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

// ✅ Name the network call.
function fetchUser(id: UserId): Promise<User> {
  return fetch(`/api/users/${id}`).then(r => r.json());
}

// ✅ `get` for the synchronous, cached, or in-memory lookup.
function getUserById(id: UserId, users: Map<UserId, User>): User | undefined {
  return users.get(id);
}
```

`get` = pure retrieval. `fetch` = network. `load` = bootstrap. Reserve each verb for one kind of work.

### A "validator" returns a verdict, not the transformed value

```ts
// ❌ Surprise — the validator also normalizes.
function validateEmail(input: string): string {
  if (!EMAIL_REGEX.test(input)) throw new Error('invalid');
  return input.toLowerCase().trim();
}

// ✅ Two clear operations. The verdict is `boolean` or a typed Result.
function isValidEmail(input: string): boolean {
  return EMAIL_REGEX.test(input);
}

function parseEmail(input: unknown): EmailAddress {
  // throws on invalid, returns branded type on valid
}
```

If the function is *really* doing both (validate + normalize + return as a branded type), it's a parse — name it `parseEmail`, not `validateEmail`.

### Side effects are in the name

```ts
// ❌ A "format" function that also logs.
function formatPrice(price: Money): string {
  console.log(`formatting price ${price.toString()}`);
  return `$${(price.amountCents / 100).toFixed(2)}`;
}

// ✅ Drop the log (it doesn't belong in formatting).
function formatPrice(price: Money): string {
  return `$${(price.amountCents / 100).toFixed(2)}`;
}

// ✅ Or, when the side effect is intended, name it.
function logAndFormatPrice(price: Money, logger: Logger): string {
  logger.debug(`formatting price ${price.toString()}`);
  return `$${(price.amountCents / 100).toFixed(2)}`;
}
```

Naming smell test: if removing the side effect doesn't change what the function "promises" to do, the side effect doesn't belong. If it does, the name needs to say so.

This is why not every "and" is a smell. `fetchAndSaveUser` hides *two unrelated operations* behind one call — split it. `checkRateLimitAndThrowError` *discloses* a guard that's part of one operation — the "and" is honesty, not a second concern. When the second clause is just a guard, a `throwIf*` / `assert*` prefix often reads better than "and".

### `find` returns optional; `findOrThrow` throws

```ts
// ❌ A `find` that throws.
async function findUserById(id: UserId): Promise<User> {
  const user = await users.findFirst({ where: { id } });
  if (!user) throw new UserNotFoundError(id);
  return user;
}

// ✅ Two names, two contracts. Caller picks.
async function findUserById(id: UserId): Promise<User | null> {
  const user = await users.findFirst({ where: { id } });
  return user ?? null;
}

async function getUserByIdOrThrow(id: UserId): Promise<User> {
  const user = await findUserById(id);
  if (!user) throw new UserNotFoundError(id);
  return user;
}
```

If your ORM already spells out the convention — Prisma's `findUnique` vs `findUniqueOrThrow`, `findFirst` vs `findFirstOrThrow` — mirror it. A hand-written `findToken` that throws while the Prisma layer right beside it carefully says `OrThrow` is the surprise: the reader trusts the suffix everywhere else and gets burned by the one place that breaks it.

The reader knows from the name which contract they're getting. No surprise.

### Async functions are honest about being async

```ts
// ❌ Returns a Promise but didn't need to.
async function formatPrice(price: Money): Promise<string> {
  return `$${(price.amountCents / 100).toFixed(2)}`;
}

// ✅ If it's not actually async, it isn't.
function formatPrice(price: Money): string {
  return `$${(price.amountCents / 100).toFixed(2)}`;
}
```

The rule is "no *needless* async," not "never async on a fast path." A function that awaits on *some* branches should stay `async` so callers always `await` one consistent shape — the surprise would be a union of `T | Promise<T>` that callers must inspect. Keep it `async` when any path is genuinely async; drop `async` only when *every* path is synchronous.

The `async` keyword tells callers "this awaits something." When it doesn't, the keyword is a lie. Callers who `await` it pay a microtask hop; callers who don't `await` it forget to handle the Promise.

### Conventional parameter order

```ts
// ❌ The "from / to" order is reversed compared to convention.
function transfer(toAccount: AccountId, fromAccount: AccountId, amount: Money) { /* ... */ }

// ✅ Caller's mental model: "from, to, amount" — match it.
function transfer(fromAccount: AccountId, toAccount: AccountId, amount: Money) { /* ... */ }
```

When a project develops a convention (callbacks last; ids first; options object last), follow it. Each deviation is a small surprise.

### Predictable return shapes

```ts
// ❌ Returns a string on success, an object on error.
function chargeCard(amount: Money): string | { error: string } { /* ... */ }

// ✅ One shape, discriminated.
function chargeCard(amount: Money): Result<ChargeId, ChargeError> { /* ... */ }
```

A return type that varies by outcome forces every caller to type-narrow at the call site. A discriminated `Result<T, E>` is consistent.

## Pressure Resistance

**"Renaming will break callers."** Use the IDE refactor. The break is one search-and-replace; the surprise is permanent until renamed.

**"The current name is too entrenched."** If the name doesn't match the behavior, every reader pays. Renaming once costs less than the perpetual confusion.

**"I'd have to split one function into two."** Yes — that's the whole point. A function called `validateAndNormalize` is doing two things. Two functions, two clear names. Or: rename to `parseEmail` if the two are really one operation.

**"It's just one extra side effect."** The first one. The second one. The third one. The naming-vs-behavior drift accumulates. Hold the line on each commit.

**"TypeScript catches the type mismatches."** TS catches *type* surprises, not *side-effect* surprises. A `formatPrice` that also logs typechecks fine; the surprise is runtime behavior.

## Common Mistakes

| Smell | Fix |
|---|---|
| Name uses "and" to join two *unrelated* operations (`fetchAndSaveUser`) | Split into two functions, two names. (An "and" that honestly *discloses a side effect* — `logAndFormatPrice`, `checkRateLimitAndThrowError` — is fine; see "Side effects are in the name".) |
| `get*` or `find*` that throws on missing data | Use `findOrThrow` / `getOrThrow`, or return `T \| null` |
| `compute*` / `format*` that mutates a parameter | Make it pure; return the new value |
| "Validator" that returns the transformed value | Rename to `parse*` — that's what it is |
| `async function` that awaits nothing | Drop the `async` keyword; return the value directly |
| `console.log` or network call inside a function whose name suggests pure computation | Remove the effect or rename the function |
| Parameters in different order from a similar function in the same module | Match the local convention |
| Return type that varies by outcome (`string \| { error }`) | Use a discriminated `Result<T, E>` |

## The Bottom Line

**A name is a promise. The body has to keep it.**

If the reader has to open the body to know what the function does, the name is wrong. Fix the name; fix the body; never let them disagree.

## Related

- `tell-dont-ask` — named operations that match intent
- `naming-ahclc` — a name that predicts behavior

## Reference

- Steve McConnell, *Code Complete* 2e (2004), ch. 7 ("High-Quality Routines") and ch. 11 ("The Power of Variable Names") — the canonical treatment of "names should match behavior."
- Robert C. Martin, *Clean Code* (2008), ch. 2 ("Meaningful Names") and ch. 3 ("Functions") — same rule, different angle.
- Joshua Bloch, *Effective Java* 3e (2018), Item 56 ("Design APIs to be hard to misuse") — the absence of surprise = the absence of misuse.
