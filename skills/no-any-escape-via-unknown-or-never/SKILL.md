---
name: no-any-escape-via-unknown-or-never
description: Use when about to type a variable, parameter, or return as `any`. Use when reviewing code that contains `: any`, `as any`, `Array<any>`, `Record<string, any>`, or `@ts-ignore`/`@ts-expect-error` without justification. Use when wrapping an untyped third-party library and the easy path is to type its surface as `any`. Use when a parsing boundary returns an unwieldy union and the temptation is to cast to `any`. Use when an unreachable branch is silenced with `as any` instead of `never`.
---

# No `any` — Escape Via `unknown` or `never`

## Overview

**`any` does not appear in application code.** When the type system cannot describe what you have, the correct escape is `unknown` followed by a narrowing parse, or `never` for branches that should not exist.

`any` disables type-checking *transitively* — every value derived from an `any` is also `any`, silently spreading through the codebase. `unknown` disables type-checking *locally* and forces the next reader to narrow before use. The difference between them is the entire point of having a type system.

## The Iron Rule

```
NEVER use `any`. Use `unknown` and narrow, or `never` for unreachable branches.
```

**No exceptions:**
- Not for "it's just temporary"
- Not for "the shape is too complex to type"
- Not for "TypeScript can't infer this"
- Not for "I'm just prototyping"

## Why

A field typed as `any`:
- Accepts every value (good).
- Returns a value the type system *believes* is every type (bad).
- Lets `x.foo.bar.baz()` typecheck even when `x` is `null` (very bad).
- Makes downstream variables typed `any` by inference (catastrophic — the rot spreads).

A field typed as `unknown`:
- Accepts every value (good).
- Returns a value the type system makes you *narrow* before using (good).
- Forces a parse step at the boundary where the value enters typed code.

`never` is the other escape: it represents *unreachable* state. Where `unknown` is "I don't know what this is yet," `never` is "this can't exist." A switch over a discriminated union whose `default` branch is reached because someone added a new variant should fail at compile time — that's what `never` is for.

## Detection

You are violating the rule if any of these are true:

- `: any` appears in a parameter, return, field, or variable annotation.
- `as any` appears anywhere outside a *single* declared boundary adapter for a typeless library.
- `Array<any>`, `Record<string, any>`, `Promise<any>`, `Map<string, any>`.
- `// @ts-ignore` or `// @ts-expect-error` without an inline reason comment naming the constraint.
- A function returns `unknown` and the caller immediately `as`-casts instead of narrowing.
- An exhaustive-switch's `default` branch handles a real future variant via `as any` cast, not `never`.
- `JSON.parse(...)` is assigned to a typed variable without going through a schema parse.
- `tsconfig.json` has `"noImplicitAny": false`.

## The Pattern

### Inbound untyped data — `unknown` then parse

```ts
// ❌ Pretending fetch gave you a User.
async function fetchUser(id: UserId): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  return res.json() as User; // `any` sneaks in via the json() return + the as cast
}

// ✅ `unknown` at the boundary; parse converts to the typed shape.
async function fetchUser(id: UserId): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  const raw: unknown = await res.json();
  return User.parse(raw); // schema; throws on bad shape
}
```

The `unknown` is invisible to consumers — they get `Promise<User>`. The discipline lives at the seam.

### Unreachable branches — `never`

```ts
type Order =
  | { kind: 'pending';   placedAt: Date }
  | { kind: 'shipped';   trackingId: string }
  | { kind: 'cancelled'; reason: string };

function summarize(order: Order): string {
  switch (order.kind) {
    case 'pending':   return `placed ${order.placedAt.toISOString()}`;
    case 'shipped':   return `shipped (track: ${order.trackingId})`;
    case 'cancelled': return `cancelled: ${order.reason}`;
    default: {
      // ❌ const _: any = order;  — silent on new variants
      // ✅ Forces a compile error when a new variant is added.
      const _exhaustive: never = order;
      throw new Error(`unhandled order kind: ${String(_exhaustive)}`);
    }
  }
}
```

When someone adds `{ kind: 'returned'; ... }` to the union, the assignment `_exhaustive: never = order` stops compiling at this exact site.

### Third-party library with no types — wrap once at the boundary

Some libraries genuinely ship without types, or with types so loose they're effectively `any`. The rule isn't "never use them" — it's "the `any` lives in *one* adapter file, and the rest of the codebase sees a typed surface."

```ts
// adapters/pdf-parser.ts — the ONE place where `any` lives.
import legacy from 'untyped-pdf-parser';
import { z } from 'zod';

const ParsedPdfSchema = z.object({
  pageCount: z.number().int().nonnegative(),
  textByPage: z.array(z.string()).readonly(),
});
type ParsedPdf = z.infer<typeof ParsedPdfSchema>;

export async function parsePdf(bytes: Uint8Array): Promise<ParsedPdf> {
  // Boundary-only `any`. Justified because the library is untyped.
  const raw: any = await legacy.parse(bytes);
  return ParsedPdfSchema.parse({
    pageCount: raw.numPages,
    textByPage: raw.pages?.map((p: any) => p.text) ?? [],
  });
}
```

Two rules at the boundary:
1. The `any` does not leak — the returned type is `ParsedPdf`, not `any`.
2. There's a runtime parse — the type assertion is *backed* by a check.

Every other file imports `parsePdf` and gets a typed contract. The rot is contained.

### `@ts-expect-error` with a reason — the only acceptable escape comment

```ts
// ✅ Justified, narrow, time-bounded.
// @ts-expect-error – upstream @types/foo@2.3 has wrong signature for `.bar`;
//                    fixed in 2.4, pinned for compat with consumer X.
foo.bar(input);

// ❌ Generic silencing.
// @ts-ignore
foo.bar(input);
```

If the situation that caused the suppression is resolved (the upstream type ships), `@ts-expect-error` becomes an error itself — it's self-cleaning. `@ts-ignore` rots forever.

### Generic functions — constrain the type variable, don't `any` it

```ts
// ❌ Defeats the type system.
function first(arr: any[]): any {
  return arr[0];
}

// ✅ Generic preserves the element type.
function first<T>(arr: readonly T[]): T | undefined {
  return arr[0];
}
```

`any[]` says "I gave up." A type variable says "I don't need to know — but my caller does."

### `unknown` in error handlers

TypeScript 4.4+ types `catch` clause variables as `unknown` (under `useUnknownInCatchVariables`). Honor it.

```ts
// ❌
try { /* ... */ } catch (err: any) {
  log.error(err.message); // crashes if err is a non-Error throwable
}

// ✅
try { /* ... */ } catch (err: unknown) {
  const message = err instanceof Error ? err.message : String(err);
  log.error(message);
}
```

The narrowing forces you to handle the "someone threw a string" case that JS allows.

### `tsconfig.json` — make the rule enforceable

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "useUnknownInCatchVariables": true,
    "noUncheckedIndexedAccess": true
  }
}
```

`strict: true` covers `noImplicitAny`. `noUncheckedIndexedAccess` makes `array[i]` return `T | undefined` instead of `T`, preventing a sneaky `any`-like hole. These settings make the rule a build-time guarantee, not a code-review habit.

## Pressure Resistance

### "It's just temporary"

Temporary `any`s become permanent. Once the rest of the codebase grows downstream of the `any`, removing it is a cross-cutting refactor — touching every consumer. The "temporary" framing predicts a tomorrow that doesn't come.

### "The shape is too complex to type"

If the shape is too complex to type, it's too complex to use safely. Parse it into a domain type at the boundary; the rest of the code consumes the domain type. The complexity is bounded to one adapter.

### "TypeScript inference can't figure this out"

Inference's gaps are not licenses for `any`. The escape is `unknown` + narrowing, or a type variable, or a more specific structural type. If inference fails on a real shape, the shape is usually under-described — naming the shape (with a type alias or interface) often unblocks inference too.

### "The library types are wrong, I'll just `any` past them"

`@ts-expect-error` with a one-line reason is the right tool. It marks the spot, names the cause, and complains when the cause goes away. `as any` silently absorbs the wrongness forever.

### "I'm just prototyping"

The rule applies to code you ship, not code you delete. Prototypes that survive past their first afternoon should be retyped before merging. A `git grep ": any"` in PR review catches the survivors.

### "Generic types are confusing"

A type variable (`<T>`) is a feature of every modern language. If `<T>` is confusing for the consumer, the API surface should narrow — not the safety. Sometimes the right answer is two specific functions instead of one generic; never one `any`-typed function.

## Red Flags

- `: any` in a function signature.
- `as any` anywhere outside an adapter file.
- `Record<string, any>` for non-telemetry data — usually want `Record<string, unknown>` or a typed shape.
- `Array<any>` or `any[]` — almost always a missed generic.
- `// @ts-ignore` with no comment.
- `// @ts-expect-error` with no explanation of *what* error and *why* it's expected.
- `JSON.parse(...)` whose result is used without a schema parse.
- A `default` switch branch using `as any` to silence the compile error.
- A function whose return is `Promise<any>` — the caller will inherit `any` transitively.

**All of these mean: replace with `unknown` + parse, a type variable, or `never`.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "TypeScript can't infer this" | Then say `unknown` and narrow. `any` is not "I can't infer"; it's "I'm not checking." |
| "It's an internal function" | Internal functions are consumed by other internal functions. The `any` propagates. |
| "I need to handle arbitrary JSON" | `unknown` + parse. The arbitrary part lives at the boundary; the body has typed data. |
| "It's just for this one line" | Lines never stay one. The inferred-`any` downstream is the real cost. |
| "Library types are wrong" | `@ts-expect-error` with a reason. Self-cleaning when the upstream fixes it. |
| "The codebase already has `any`s" | The existing ones are debt. New ones add to the debt. Stop the bleed; pay it down opportunistically. |
| "Generics are too abstract" | A type variable is less abstract than `any` — it's a specific promise. `any` is *more* abstract because it promises nothing. |

## Reference

- The TypeScript team's [Do's and Don'ts](https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html) — official: prefer `unknown` over `any` when the type is genuinely unknown.
- Dan Vanderkam, *Effective TypeScript* 2e (2024), item 62 ("Avoid `any`"), item 49 ("Prefer `unknown` to `any`").
- TypeScript 3.0 release notes (2018) — Anders Hejlsberg's original `unknown` proposal: `unknown` is the type-safe counterpart of `any`. Same accept-anything semantics, opposite use semantics.
