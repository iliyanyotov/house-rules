---
name: prefer-type-inference-annotate-at-boundaries
description: Use when about to write a type annotation on a local variable that TypeScript would infer correctly. Use when reviewing code where every `const` has a redundant `: string` after it. Use when a refactor changes a function's return type and ten callers have to be updated because they all annotated `: OldType` instead of relying on inference. Use when deciding whether to add an explicit return type to an exported function. Use when reading code where the noise of annotations obscures the logic.
---

# Prefer Type Inference; Annotate at Boundaries

## Overview

**Let TypeScript infer types in the interior. Annotate at module boundaries** — public function signatures (parameters and return), exported `const` shapes, and the places where data crosses from `unknown` into the type system.

The rule has two halves and both matter: *use inference where it works* (most of the body of most functions), and *insist on annotations where it doesn't* (parameters, returns, and exported shapes — the contracts the rest of the codebase depends on).

## The Iron Rule

```
NEVER annotate what inference already gives you. ALWAYS annotate the boundary the rest of the codebase reads.
```

**No exceptions:**
- Not for "explicit is always better"
- Not for "annotations document the code"
- Not for "I want to lock the type down"
- Not for "if inference is wrong I'll fix it"

## Why

Inference is one of TypeScript's strongest features. The compiler reads `const user = await fetchUser(id)` and knows `user`'s exact type — sharper than most humans would write by hand. When the source function's return type changes, the local variable's inferred type changes with it. No update needed. The refactor flows downstream.

Manually annotating an interior local — `const user: User = await fetchUser(id)` — *freezes* the local type at the moment you wrote it. If `fetchUser` later returns `User | null`, your annotation is now stale, and the compiler accepts it because of subtyping rules.

But inference is also dangerous *at boundaries*. An exported function without a return-type annotation has its public contract determined by *whatever the last line of the body inferred to*. Refactor the body, and you've silently changed the public type. Every caller is now compiling against a new contract without anyone noticing.

The discipline splits cleanly:

- **Interior:** trust inference. Less noise, faster refactors.
- **Boundary:** annotate. Pin the contract; make changes to it require deliberate edits.

## Detection

You are violating the rule if any of these are true:

- Every local `const` and `let` has a type annotation.
- A `const x: string = 'hello'` or `const n: number = 42` appears in code.
- An *exported* function has no return type annotation — its contract is implicit.
- A callback parameter is annotated (`arr.map((item: User) => ...)`) when the array's element type would have flowed in.
- A type annotation matches the immediately-following initializer exactly (the annotation is restating the inference).
- A type annotation drifts from what's actually assigned (the annotation is a *lie* — the compiler accepts it because of subtyping, but the runtime value is narrower or different).
- An interior annotation *widens* the inferred type (`const x: SomeEnum = 'LITERAL'` where inference would give the literal). Unless something downstream needs the wider type, it's noise — delete it; if widening is intentional, use `satisfies` or let the consumer's parameter type do the widening.
- A refactor changing a function's return type forced you to update dozens of call sites' local annotations.

## The Pattern

### Interior — let inference do its job

```ts
// ❌ Noise — every annotation restates what inference already gives.
async function getDisplayName(userId: UserId): Promise<string> {
  const user: User = await fetchUser(userId);
  const name: string = user.fullName ?? user.email;
  const trimmed: string = name.trim();
  return trimmed;
}

// ✅ Inference handles the interior; the signature is annotated.
async function getDisplayName(userId: UserId): Promise<string> {
  const user = await fetchUser(userId);
  const name = user.fullName ?? user.email;
  return name.trim();
}
```

The second form is shorter, identical in safety, and *follows refactors automatically*. If `fetchUser` later returns `User & { plan: Plan }`, the interior just absorbs the change. No edits.

### Boundary — annotate the signature

```ts
// ❌ No return annotation — public contract is whatever inference produces.
export async function fetchUser(id: UserId) {
  const [row] = await users.findById(id);
  return row;  // is this `User`? `User | undefined`?
}

// ✅ Explicit signature — the public contract is pinned.
export async function fetchUser(id: UserId): Promise<User | undefined> {
  const [row] = await users.findById(id);
  return row;
}
```

The annotation is also documentation. A caller skimming the signature sees `Promise<User | undefined>` and knows to handle the `undefined` case without reading the body.

### Callbacks — usually no annotation needed

```ts
// ❌ Restating what the array's element type already provides.
const names = users.map((user: User): string => user.name);

// ✅ Inference flows from `users: User[]` through the callback.
const names = users.map((user) => user.name);
```

The most frequent real case is over-annotating a *primitive* param the source already guarantees:

```ts
// ❌ `Object.keys()` already returns `string[]` — the annotation restates it.
Object.keys(result).map((key: string) => result[key]);

// ✅ Inference flows from `Object.keys`'s return type.
Object.keys(result).map((key) => result[key]);
```

### State initializers — annotate when initial value is too narrow

```ts
// ❌ Initial value already implies the type.
const [count, setCount] = useState<number>(0);

// ✅ Inference suffices.
const [count, setCount] = useState(0);

// ❌ Inferred too narrow — TS infers `null`, but you want `User | null`.
const [user, setUser] = useState(null);

// ✅ Pinned union — inference can't widen from `null` alone.
const [user, setUser] = useState<User | null>(null);
```

Rule of thumb: if inference produces the right type, don't annotate. If it produces something narrower than you intend (typical with `null`, empty arrays `[]`, empty objects `{}`), annotate.

### Exported `const` — annotate when intent is wider than initializer

```ts
// ❌ Inferred type is too narrow — `readonly ['active', 'paused']` instead of `OrderStatus[]`.
export const FILTERABLE_STATUSES = ['active', 'paused'];

// ✅ Annotation pins the broader intent.
export const FILTERABLE_STATUSES: readonly OrderStatus[] = ['active', 'paused'];

// ❌ Inferred type is too wide — `string`, but you wanted the literal union.
export const DEFAULT_LOCALE = 'en-US';

// ✅ `satisfies` gives both — narrow type, plus check.
export const DEFAULT_LOCALE = 'en-US' satisfies Locale;
```

### Schemas and parsers — publish the inferred type

```ts
// Parse step gives the type; export both schema and type.
export const User = z.object({
  id: UserId,
  email: z.string().email(),
  createdAt: z.date(),
});

// ✅ Use `z.infer` to derive the type — never duplicate the shape manually.
export type User = z.infer<typeof User>;
```

This is the *one* case where you write `type User = ...` despite a function-like shape — because the schema and the type are the same fact, and `z.infer` enforces it.

### Discriminated unions in state — annotate at the call site

```ts
type SubmitState =
  | { kind: 'idle' }
  | { kind: 'submitting' }
  | { kind: 'success'; message: string }
  | { kind: 'error'; message: string };

const [state, setState] = useState<SubmitState>({ kind: 'idle' });
```

The annotation is necessary — `useState({ kind: 'idle' })` alone would infer a non-discriminated `{ kind: 'idle' }`, and the next call `setState({ kind: 'submitting' })` would error. Pinning the union at the call site tells inference what's coming.

## Pressure Resistance

### "Explicit types are always better — they're documentation"

For *boundary* types, yes. For *interior* locals, no — they're noise, and worse, they're lies waiting to happen. The compiler's inferred type is the *real* type; an annotation that matches it is redundant; an annotation that doesn't match it is a bug. The next reader's IDE shows the inferred type on hover; the annotation isn't doing documentation work, it's adding extra characters between them and the logic.

### "If we annotate everything, refactors are safer"

Backwards. Annotations *pin* the type at the annotation site. When the source changes, an interior annotation either silently continues to typecheck (lie) or noisily fails everywhere it appears (cost). Inference *follows* the source change with zero edits. Boundary annotations *should* fail loudly when the source changes — that's their job. Interior annotations duplicating inference don't help.

### "Without return-type annotations, refactors can change the public type silently"

True — that's why the *boundary* half of this rule exists. Always annotate exported function return types. Inside the body, let inference handle it.

### "It's a trivial one-liner — the return type is obvious"

Triviality is about the *body*, not the *contract*. A one-line exported function still exposes a public type callers compile against: `return isPrismaObj(obj) ? obj : undefined` silently exports `JsonObject | undefined`, and a one-character body edit can change that exported type with no warning. Annotate the return regardless of body length — the length of the body has nothing to do with the cost of the contract changing.

### "Inference fails on complex shapes"

Sometimes. When inference produces the wrong shape (too wide, too narrow, missing a generic constraint, picking the wrong overload), that's the *exact* place to annotate. The rule isn't "never annotate"; it's "annotate where inference doesn't give you what you mean."

### "What about `noImplicitAny`?"

`noImplicitAny` requires *parameter* annotations — exactly the boundary case this rule says to annotate. The rule doesn't conflict; it amplifies the same instinct. Parameters: yes. Local variables initialized from typed values: no.

## Red Flags

- `const x: T = ...` where `T` is exactly what `...` would infer.
- A callback parameter annotation that restates the array's element type.
- `useState<X>(initialValueWithObviousType)`.
- An *exported* function with no return-type annotation.
- Every line in a function body has a type annotation; the file's signal-to-noise is low.
- A refactor PR with dozens of "update annotation" edits in call sites — the annotations weren't pulling weight.
- An annotation that doesn't match the actual runtime shape (subtle bug: `string` annotated, `string | null` returned).

**All of these mean: either delete the redundant annotation, or move it to the boundary where it earns its space.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Explicit = clearer" | Explicit annotations duplicate inference; readers see the inferred type on hover. The duplication adds noise, not signal. |
| "Annotations document the code" | Boundary annotations document the contract. Interior annotations document the obvious. |
| "I want to lock the type down" | Locking the type down is exactly what makes refactors painful when the source legitimately changes. Lock the *boundary*, let the interior float. |
| "If inference is wrong I'll fix it" | You may not notice. The annotation matching inference today protects you from a silent change tomorrow — except it doesn't, because the annotation isn't a check, it's a duplicate. |
| "Annotations help juniors read the code" | Juniors read the hover. Annotations that lie when the source changes are *worse* for juniors. |
| "It's just a few extra characters" | A few extra characters × every line × every file = noticeable noise. And every one is a place for the annotation to drift from the truth. |

## Related

- `parse-dont-validate` — boundaries are annotated; the interior infers from the proof the boundary carries

## Reference

- Google TypeScript Style Guide, [§Type System / Type Inference](https://google.github.io/styleguide/tsguide.html#type-inference) — *"Rely on the type inference of the compiler as much as possible. Omit type annotations for trivially inferred types."*
- Dan Vanderkam, *Effective TypeScript* 2e (2024) — item 18 (*Use `typeof` to Infer Types*), item 19 (*Avoid Cluttering Your Code with Inferable Types*), item 9 (*Prefer Type Annotations to Type Assertions*).
- TypeScript Handbook, ["Type Inference"](https://www.typescriptlang.org/docs/handbook/type-inference.html) — the official position is that inference is preferred where it gives the right answer.
