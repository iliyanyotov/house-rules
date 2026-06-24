---
name: make-illegal-states-unrepresentable
description: Use when designing state shapes for components, server responses, form state, or domain entities. Use when tempted to model state with parallel booleans (`isLoading`, `isError`, `isSuccess`), optional fields that should travel together, or "either/or" data encoded as two nullable fields. Use when reviewing a type that has more impossible combinations than possible ones. Use when a comment says "only set when X is true."
---

# Make Illegal States Unrepresentable

## Overview

**Model state as a discriminated union of legal shapes.** Never as parallel booleans, parallel optionals, or "this field is only set when that other field is true."

If two fields must travel together, put them in the same variant. If two fields are mutually exclusive, put them in different variants.

## The Iron Rule

```
NEVER model state as parallel booleans or correlated optionals. Use a discriminated union.
```

**No exceptions:**
- Not for "the library returns this shape"
- Not for "booleans are simpler to read"
- Not for "I might need more states later"
- Not for "it's more code"

## Why

A type with three booleans has eight combinations. Usually four to six of them are bugs. Every consumer must defend against impossible combinations — every component, every render, every test.

A discriminated union has exactly as many shapes as there are legal states. The compiler enforces it. Impossible combinations don't exist at the type level, so no consumer has to defend against them. You're not adding a runtime check; you're deleting an entire class of bugs.

## Detection

You are violating the rule if any of these are true:

- A type has `isLoading`, `isError`, `data?`, `error?` as siblings.
- A type has `success: boolean` paired with `data?` and `error?` where only one is meant to be set.
- A component reads `if (loading) ...; if (error) ...; if (data) ...` and the branches aren't exhaustive.
- A form state has `editingId: string | null` paired with `editingDraft: T | null` that must be set/unset together.
- A type description uses the words "if X then Y is set" — that's the rule that should be in the type.
- A component with multiple correlated `useState` calls: `isSubmitting`, `isSubmitted`, `submitStatus` — the illegal combos (`isSubmitting && isSubmitted`) are bugs waiting to surface.

## The Pattern

### Fetch state — the canonical example

```ts
// ❌ Eight combinations. Four are legal. Four are bugs waiting.
type FetchState<T> = {
  isLoading: boolean;
  isError: boolean;
  data: T | null;
  error: Error | null;
};

// ✅ Exactly four states. The compiler enforces it.
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };
```

The consumer becomes obvious — no `state.data?.map(...)`, no `if (state.data) { ... }`:

```tsx
function OrderList({ state }: { state: FetchState<Order[]> }) {
  switch (state.status) {
    case 'idle':    return <EmptyState />;
    case 'loading': return <Skeleton />;
    case 'success': return <List items={state.data} />;     // `data` exists by type
    case 'error':   return <ErrorBanner error={state.error} />; // `error` exists by type
  }
}
```

The narrowing comes from the discriminant. Impossible branches don't exist.

### Form submission — the most common offender

```ts
// ❌ Three useState, eight combinations, half are bugs.
const [isSubmitting, setIsSubmitting] = useState(false);
const [isSubmitted, setIsSubmitted] = useState(false);
const [submitStatus, setSubmitStatus] = useState<{
  type: 'success' | 'error' | null;
  message: string;
}>({ type: null, message: '' });

// Illegal combos the compiler can't catch:
//   isSubmitting && isSubmitted
//   submitStatus.type === 'success' && !isSubmitted
//   isSubmitted && submitStatus.type === null

// ✅ One useState. Four states. Compiler enforces the rest.
type SubmitState =
  | { kind: 'idle' }
  | { kind: 'submitting' }
  | { kind: 'success'; message: string }
  | { kind: 'error'; message: string };

const [state, setState] = useState<SubmitState>({ kind: 'idle' });
```

The transition `idle → submitting → success | error` is encoded in the type. The component cannot render in an impossible state because impossible states don't exist.

Collapse *correlated* state, not *orthogonal* state. In a multi-step form the wizard step is its own union (`WizardStep`) and the submit lifecycle is a *second* union (`SubmitState`) — they vary independently, so forcing them into one union creates impossible-product noise, not safety. The rule is "one union per axis of variation," not "one union per component."

### "Either A or B" — never two correlated optionals

```ts
// ❌ Both can be set. Both can be unset. The compiler can't help.
type Payment = {
  cardToken?: string;
  bankAccountId?: string;
};

// ✅ Exactly one. The compiler enforces it.
type Payment =
  | { method: 'card'; cardToken: string }
  | { method: 'bank'; bankAccountId: string };
```

### Result types — discriminate, don't double-optional

```ts
// ❌ Caller must decide which field to read based on which is null.
type Result<T> = { data: T | null; error: string | null };

// ✅ Caller pattern-matches.
type Result<T, E = string> =
  | { ok: true; data: T }
  | { ok: false; error: E };
```

The boundary returns a `Result`; the consumer pattern-matches once. No `if (result.data)` defensive chains downstream.

## Pressure Resistance

### "But fetching libraries return `isLoading`, `data`, `error` — that's the pattern"

That's their *projection* for ergonomic destructuring, and they also expose a `status` discriminant for exactly this reason. Internally, those libraries model the state as a discriminated union; the parallel booleans are derived. When *you* design types, model the union. When you consume their types, use the `status` field.

### "Discriminated unions are awkward to spread or partially update"

That awkwardness is the *point*. A "partial update" of a `loading → success` transition is replacing one shape with another, not patching a field. If a reducer wants to spread, it's working with parallel booleans by another name.

### "It's more code"

Compare line for line: a 4-field type with ~6 illegal combinations (every consumer defends) vs. a union with 4 variants and 0 illegal states (every consumer pattern-matches). Line count is similar. Bug surface is not.

### "JSON doesn't understand unions"

JSON understands objects with discriminant fields fine. `{ status: 'loading' }` serializes. So does `{ status: 'success', data: ... }`. Your wire format and your type can be the same union.

### "I need this to interop with a flag-based API"

Convert at the boundary. The external shape is wire-level; your type is domain-level. The conversion is one function, in one place.

## Red Flags

- Three or more boolean fields named in the same type
- A type with `?` on more than two sibling fields
- A comment like `// only set when foo is true`
- A test that asserts a "shouldn't happen" combination doesn't crash
- A switch on one field plus an `if` on another inside the same branch
- The phrase "let me also handle the case where both are null" in code review

**All of these mean: the type has illegal states. Collapse into a union.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The library returns this shape, so I have to model it this way" | Convert at the boundary. Library shape is wire-level; your type is domain-level. |
| "Booleans are simpler to read" | Until two of them are true. |
| "I might need more states later" | Add a variant. That's the whole point of unions. |
| "It's hard to construct intermediate states" | Intermediate states are themselves variants. If you can't name one, you don't have it. |
| "Unions don't compose well" | They compose via union: `A | B | C`. They don't compose via spread — by design. |
| "It's faster to write four booleans" | And slower to debug for the rest of the type's life. |

## Related

- `exhaustive-switch` — discriminated unions define legal states; exhaustive switch enforces handling them
- `parse-dont-validate` — parsing produces the narrowed states; illegal states never enter typed code

## Reference

- Yaron Minsky, ["Effective ML"](https://blog.janestreet.com/effective-ml-revisited/) (Jane Street, 2011) — the canonical phrasing: *"Make illegal states unrepresentable."* The OCaml community lore that crossed into TypeScript via discriminated unions.
- Richard Feldman, ["Making Impossible States Impossible"](https://www.youtube.com/watch?v=IcgmSRJHu_8) (Elm Conf, 2016) — the most-cited modern restatement, applied to UI state.
