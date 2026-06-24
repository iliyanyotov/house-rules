---
name: satisfies-over-as
description: Use when reaching for `as` to convince the TypeScript compiler. Use when declaring a constant config object, a literal lookup table, a const tuple, or a default-value object. Use when the compiler complains about a literal you "know" is correct. The answer is almost never `as`; it's `satisfies`, or a fix to the type.
---

# `satisfies` over `as`

## Overview

**Use `satisfies` to check a value against a type without widening or coercing.** `as` is reserved for two cases: branding a parsed value at a boundary, and narrowing a verified `unknown`. Anywhere else, `as` is a lie to the compiler.

`as` widens (and lies if needed). `satisfies` narrows (and refuses to lie).

## The Iron Rule

```
NEVER use `as` to silence the compiler. Use `satisfies` to check, or fix the type.
```

**No exceptions:**
- Not for "I know the type is correct, the compiler is just being annoying"
- Not for "it's a one-liner test, who cares"
- Not for "external library types are wrong, I have to `as`"
- Not for "I want to pass `Partial<Foo>` as `Foo`"

## Why

`as` is the type system's escape hatch. It tells the compiler "trust me." When the compiler trusts you and you're wrong, the bug ships — there is no second check. `as` defeats the entire reason TypeScript exists.

`satisfies` is the inverse: it verifies that a value *conforms to* a type, **while preserving the value's narrower literal type**. You get the safety of an annotation and the precision of an inference.

If `satisfies` doesn't accept your value, the value is wrong or the type is wrong. Fix one of them.

## Detection

You are violating the rule if any of these are true:

- A line contains `as Something` and the source is not `unknown` from a parser or boundary.
- A function returns `x as Foo` because the inferred type was "close enough."
- A const declaration uses `: Foo = ...` to satisfy a type but loses literal-type information you actually need.
- A `Record<string, Foo>` is written when a known set of keys is being defined — kills key-level autocompletion.
- `as unknown as Foo` (the double-cast) appears anywhere — that's an alarm, not a pattern.

Note: `as const` is a different operator. It freezes a literal's type and is *fine* — this rule is about `as Type`.

## The Pattern

### Const objects — preserve literals with `satisfies`

```ts
// ❌ Annotated: keys widen to `string`, values widen to the type.
const ROLES: Record<string, { label: string; rank: number }> = {
  admin:  { label: 'Administrator', rank: 100 },
  member: { label: 'Member', rank: 10 },
  guest:  { label: 'Guest', rank: 0 },
};
// `ROLES.admin` is `{ label: string; rank: number } | undefined` — worse.
// You lose autocomplete on `ROLES.foo`.

// ❌ Asserted: lies about the type, drops checks.
const ROLES = {
  admin:  { label: 'Administrator', rank: 100 },
  member: { label: 'Member', rank: 10 },
  guest:  { label: 'Guest', rank: 0 },
} as Record<'admin' | 'member' | 'guest', RoleDef>;
// `ROLES.admin.label` is `string`, not `'Administrator'`.

// ✅ Satisfies: checks against the type, keeps the literal precision.
type RoleDef = { label: string; rank: number };
const ROLES = {
  admin:  { label: 'Administrator', rank: 100 },
  member: { label: 'Member', rank: 10 },
  guest:  { label: 'Guest', rank: 0 },
} satisfies Record<string, RoleDef>;

// `ROLES.admin.label` is `'Administrator'` (literal).
// `ROLES.foo` is a compile error.
// `RoleDef` is enforced on every entry.
```

### Mutable accumulators — annotate the type parameter, not `satisfies`

`satisfies` is for *finished, literal* values. A mutable accumulator that you build up with index assignment is the opposite case — and the fix is a type *parameter*, not `satisfies` or `as`:

```ts
// ❌ `{} as Record<string, AppMeta>` — asserts a shape the empty object doesn't have.
// ❌ `{} satisfies Record<string, AppMeta>` — narrows to `{}`, so `store[key] = …` won't typecheck.
// ✅ Annotate reduce's type parameter; the accumulator stays wide enough to assign into.
const map = items.reduce<Record<string, AppMeta>>((store, key) => {
  store[key] = deriveMeta(key);
  return store;
}, {});
```

`satisfies {}` fails here because it narrows the accumulator to exactly `{}`, rejecting the index assignment. The type argument on `reduce<...>` is the right tool — it widens the accumulator without an `as`.

```ts
// ❌ Pretending the inferred shape matches.
function buildEvent(): OrderEvent {
  return { kind: 'created', orderId: id } as OrderEvent;
}

// ✅ Annotate the return type. Let the compiler verify.
function buildEvent(): OrderEvent {
  return { kind: 'created', orderId: id };
}
```

If the compiler complains here, the bug is real. `as` would have hidden it.

The same trap hides in multi-branch arrows:

```ts
// ❌ Each branch coerces separately; nothing is verified.
const toEvents = (rows: Row[]) =>
  rows.length === 0 ? ([] as OrderEvent[]) : rows.map((r) => ({ kind: r.k, id: r.id }) as OrderEvent);

// ✅ Annotate the return; both branches — and the `.map` body — are now checked.
const toEvents = (rows: Row[]): OrderEvent[] =>
  rows.length === 0 ? [] : rows.map((r) => ({ kind: r.k, id: r.id }));
```

The fix is the return annotation, not deleting each `as` in place — the annotation is what forces every branch to be verified.

### Config arrays with discriminated unions

```ts
type RouteConfig =
  | { method: 'GET';  path: string; handler: GetHandler }
  | { method: 'POST'; path: string; handler: PostHandler };

const routes = [
  { method: 'GET',  path: '/users', handler: listUsers },
  { method: 'POST', path: '/users', handler: createUser },
] satisfies RouteConfig[];

// Each entry is narrowed to its variant — `routes[0].handler` is `GetHandler`.
// `routes[1].handler` is `PostHandler`. With `as RouteConfig[]`, both widen.
```

### The legitimate uses of `as`

**1. Branding a parsed value at the boundary:**

```ts
export const UserId = {
  parse: (raw: unknown): UserId => cuid.parse(raw) as UserId,
};
```

The `as` here is the proof tag. The schema parse did the work; `as` records the result. One place per brand. Reviewed.

**2. Narrowing after a verified runtime check (rare; usually a code smell):**

```ts
// querySelector returns Element | null. Verify with instanceof.
const input = form.querySelector('input[name=email]');
if (!(input instanceof HTMLInputElement)) throw new Error('expected input');
// `input` is now narrowed to HTMLInputElement — no `as` needed.
```

Even here, `instanceof` did the narrowing. `as HTMLInputElement` would be redundant *and* unsafe. If you find yourself wanting `as`, look for the missing guard.

**3. `Object.keys` and its loose return type.** TypeScript intentionally returns `string[]` from `Object.keys(x)` (an object typed `{ a: 1 }` may have *extra* keys at runtime). For closed-shape config objects you control, the narrower cast is the established workaround:

```ts
const SORT_KEYS = { 'name-asc': 1, 'name-desc': 2, 'date-desc': 3 } as const;
type SortKey = keyof typeof SORT_KEYS;

const allKeys = Object.keys(SORT_KEYS) as SortKey[];
```

This is *not* a violation. Same applies to `Object.entries` and `Object.fromEntries`.

## Pressure Resistance

### "I know the type is correct, the compiler is just being annoying"

If you know it's correct, prove it with the type system. `satisfies` is that proof. If `satisfies` doesn't accept the value, you didn't know it was correct — you assumed.

### "It's a one-liner test, who cares"

Tests are where untyped bugs first show. A test with `as Result` runs against a fixture whose shape doesn't match the production type and "passes." The bug ships.

### "External library types are wrong, I have to `as`"

Wrap the library at a single seam. The wrapper does the `as` once, with a comment naming the library bug. Internal code calls the wrapper. The lie is isolated.

### "It's `as const`, isn't that fine?"

Yes — `as const` is a different operator. It freezes the value's literal type. This skill is about `as Type` assertions. Use `as const` freely on literals; combine with `satisfies` for the strongest signal.

### "I want to pass `Partial<Foo>` as `Foo`"

That's the bug, not the fix. If a function expects `Foo` and the caller has `Partial<Foo>`, either the function signature is wrong or the caller hasn't built a complete value. `as Foo` smuggles incomplete data into a contract that promises complete data.

## Red Flags

- `as Foo` where the source is already typed (not `unknown`).
- `as unknown as Foo` — the double-cast is an admission that the compiler refused once.
- A `Record<string, T>` declaration where the keys are known at write time.
- A long const config object that uses `: Type` annotation and loses autocomplete.
- A test fixture cast as the production type.
- The pattern `return ... as ReturnType` instead of annotating the function's return.
- The phrase "TypeScript is wrong here" — TypeScript is occasionally wrong; you are occasionally wrong; usually you are.

**All of these mean: the `as` is hiding a check — switch to `satisfies` or fix the type.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's faster to write" | `satisfies T` and `as T` are the same number of characters. |
| "It only matters for libraries" | Argument-swap and shape-mismatch bugs are equal in app code. |
| "The runtime is correct, the type is just decoration" | Then why have types at all? Either commit or remove. |
| "I'll come back and fix it later" | "Later" is the bug report. |
| "`as` is shorter than building the right type" | Building the right type is the work. Skipping it isn't a shortcut; it's a deferral with interest. |
| "It's an internal-only type, who cares" | Future-you cares when refactoring across the seam where the lie lives. |

## Related

- `no-any-escape-via-unknown-or-never` — sibling discipline: resist type-escape hatches
- `branded-ids` — the one legitimate `as`: branding a value after it parses

## Reference

- `satisfies` shipped in TypeScript 4.9 (November 2022). [TypeScript 4.9 release notes](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-4-9.html) — the operator's introduction and rationale.
- Matt Pocock, [Total TypeScript](https://www.totaltypescript.com/) — the clearest community framing: *"`satisfies` checks without widening, `as` widens without checking."*
