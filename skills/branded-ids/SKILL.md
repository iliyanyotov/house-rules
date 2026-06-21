---
name: branded-ids
description: Use when defining a domain entity, declaring a function that takes IDs as parameters, writing a schema, or accepting a `slug` parameter in a handler. Use when a function signature has two or more `string` parameters that represent different things — `userId`, `orgId`, `invoiceId`, `slug`. Use when a bug was caused by passing the wrong ID to the right slot. Use when a query helper accepts a raw `string` for an entity identifier.
---

# Branded IDs

## Overview

**Every domain identifier gets a nominal brand.** Functions accept branded types, never raw `string` or `number`. The only place a raw string becomes an ID is at the boundary where it was parsed.

`UserId` is not `OrgId` even though both are strings. The compiler must enforce this.

## The Iron Rule

```
NEVER pass a raw `string` or `number` where a domain ID is expected. Brand at the boundary; trust inside.
```

**No exceptions:**
- Not for "it's just a string, this is overengineering"
- Not for "I'll be careful about argument order"
- Not for "I'll add brands later when the codebase grows"
- Not for "third-party libraries return raw strings, this gets ugly"

## Why

TypeScript's type system is structural by default. `type UserId = string` is *the same type as* `string`. Pass a `userId` where the function expects an `orgId` and the compiler smiles politely while the bug ships.

Branding adds a phantom tag to the type. `UserId` becomes `string & { readonly __brand: 'UserId' }` — structurally distinct from any other branded string. The runtime is still a plain string; the compile-time identity is unique.

The result: argument-swap bugs become impossible. The compiler refuses to call `transfer(fromAccountId, toUserId)` if the parameters expect `AccountId`s. You cannot ship the bug.

## Detection

You are violating the rule if any of these are true:

- A function signature has two `string` parameters whose order matters semantically.
- A database row type uses `string` for foreign keys.
- A test fixture passes `'user_123'` directly to a function expecting an ID.
- A function called `getInvoice(id: string)` accepts an `OrgId` and silently returns nothing.
- The phrase "make sure to pass the right ID" appears in any code comment.

## The Pattern

### The brand helper — ship once

```ts
declare const brand: unique symbol;
export type Brand<T, B extends string> = T & { readonly [brand]: B };
```

Two lines. Zero runtime cost. The `unique symbol` makes the brand uncolonisable from outside the file.

### Defining branded IDs

```ts
import type { Brand } from './brand';

export type UserId = Brand<string, 'UserId'>;
export type OrgId = Brand<string, 'OrgId'>;
export type AccountId = Brand<string, 'AccountId'>;
```

### Minting at the boundary

```ts
import { z } from 'zod';

const cuid = z.string().regex(/^[a-z0-9]{24,32}$/);

export const UserId = {
  parse: (raw: unknown): UserId => cuid.parse(raw) as UserId,
};

export const OrgId = {
  parse: (raw: unknown): OrgId => cuid.parse(raw) as OrgId,
};
```

This is the *only* place `as UserId` appears. It's the boundary; the assertion is the proof that parsing happened.

### Using branded IDs

```ts
export async function transfer(
  from: AccountId,
  to: AccountId,
  initiatedBy: UserId,
  amountCents: number,
) {
  /* ... */
}

// Caller — boundary parse, then types flow.
export async function handleTransferRequest(body: unknown) {
  const input = TransferInput.parse(body);
  await transfer(
    AccountId.parse(input.from),
    AccountId.parse(input.to),
    UserId.parse(input.initiatedBy),
    input.amountCents,
  );
}

// This won't compile — argument swap caught at compile time:
//   await transfer(initiatedBy, from, to, amountCents);
//                  ^^^^^^^^^^^^
//   Argument of type 'UserId' is not assignable to parameter of type 'AccountId'.
```

### One identity, end to end

The same `UserId` should survive the whole round-trip — minted, validated, stored, and read back without decaying to `string`. Four moving parts, one type:

```ts
// 1. The primitive (the `Brand<T, B>` defined above — ship it once, reuse everywhere).
export type UserId = Brand<string, 'UserId'>;

// 2. A typed generator mints prefixed IDs already typed to the brand.
const PREFIX = { user: 'u', order: 'o', invoice: 'in' } as const;
type Entity = keyof typeof PREFIX;
type IdOf<E extends Entity> = E extends 'user' ? UserId : Brand<string, E>;

export const idGenerator = {
  id: <E extends Entity>(entity: E): IdOf<E> =>
    `${PREFIX[entity]}_${crypto.randomUUID()}` as IdOf<E>,
};

const newUser = idGenerator.id('user'); // typed UserId, value "u_…"

// 3. The parse boundary brands via a transform — the ONE legit `as`, right after validation.
export const userIdSchema = z
  .string()
  .regex(/^u_[0-9a-f-]{36}$/)
  .transform((s) => s as UserId);

// 4. The DB column TYPE is the brand, so reads come back branded.
export const users = pgTable('users', {
  id: text('id').$type<UserId>().primaryKey(),
});
```

Now one identity flows across the stack — `UserId` at every hop, never `string`:

```ts
// HTTP → service → DB column → back out. No re-casting in between.
async function handleGetUser(rawId: string) {
  const userId = userIdSchema.parse(rawId); // string → UserId, validated once
  const user = await loadUser(userId);      // UserId in
  return user.id;                           // UserId out (column type carried it)
}

async function loadUser(id: UserId): Promise<{ id: UserId }> {
  return db.query.users.findFirst({ where: eq(users.id, id) });
}
```

The brand is minted in exactly two named boundaries — the generator and the transform. Everywhere else `UserId` flows untouched. Swap `UserId` for `OrderId` at any hop and the compiler stops you.

## Pressure Resistance

### "It's just a string, this is overengineering"

The bug it prevents is a junior dev passing `userId` where `orgId` was expected, finding no rows, and shrugging. That bug has cost real money in real codebases. The brand costs two lines per ID and zero runtime overhead.

### "I'll just be careful about argument order"

You will. Until you refactor. Until the function adds a third ID parameter. Until a copy-paste from a sibling function lands the wrong variable in the wrong slot. The compiler is the only reviewer that doesn't get tired.

### "I need to log/serialize them — branding breaks that"

It doesn't. Branded values are `string` at runtime. `JSON.stringify`, `console.log`, template literals — all work unchanged. The brand only exists in types.

### "Now I have to convert at every boundary"

You convert *once* at the system boundary. Inside the system, the brand flows. If you find yourself converting in two places for the same input, you have two boundaries — that's a design issue, not a brand issue.

### "Third-party libraries return raw strings, this gets ugly"

Translate at the seam. Inside your code, types are branded. At the library boundary, you `parse` or `as`-cast — once, in a named place. The library doesn't know about your domain; that's correct.

## Red Flags

- A function signature: `(a: string, b: string, c: string) => ...`.
- A schema column with no type annotation for an ID column.
- A test that passes `'abc123'` as both a `userId` and an `orgId` and is happy.
- A bug ticket with the phrase "wrong ID was passed."
- `as` used anywhere that isn't a parser or boundary file.
- A `string` parameter named `id` with no hint of which entity it identifies.

**All of these mean: the brand is missing — declare the type and parse at the boundary.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "TypeScript should support nominal types natively" | It doesn't. Brands are the idiomatic workaround. Move on. |
| "We'll add brands later when the codebase grows" | "Later" is after the first argument-swap bug. The cost compounds. |
| "Branding doesn't work with `JSON.parse` results" | `JSON.parse` returns `unknown`; you parse through a schema and brand in the parser. Same path as any other boundary. |
| "Equality comparisons get awkward" | `userId === otherUserId` works fine — both are `UserId`. Comparing a `UserId` and an `OrgId` is a *bug*; the compiler refusing it is the feature. |
| "I want one ID type for the whole app" | Then you don't have IDs, you have strings. Re-read the failure mode. |

## Related

- `value-objects-for-domain-primitives` — branded IDs are the lightweight form of value objects
- `parse-dont-validate` — the brand is minted at the parse boundary, then trusted inward

## Reference

- Dan Vanderkam, *Effective TypeScript* 2e (2024), item 37 ("Use Branded Types for Nominal Typing") — the canonical TS treatment.
- Benjamin Pierce, *Types and Programming Languages* (2002), ch. 19 — the original "nominal vs structural" type-theory distinction the pattern stands on.
