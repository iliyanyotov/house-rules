---
name: exhaustive-switch
description: Use when writing a `switch` over a discriminated union, an enum, a string-literal type, or any closed set of values. Use when the linter complains about a missing `default`. Use when reviewing a pattern-match that "shouldn't reach this case" — that's exactly the case to fail-closed on.
---

# Exhaustive Switch

## Overview

**Every `switch` over a closed set of values ends with a `never` check.** Adding a new variant must be a compile error in every consumer until they handle it.

No silent `default`. No silent fall-through. No "we'll never hit this" comment.

## The Iron Rule

```
NEVER let a switch over a closed set silently fall through. End with assertNever, or rely on a typed return.
```

**No exceptions:**
- Not for "I'll never add a new variant"
- Not for "the linter already catches missing cases"
- Not for "my union has 20 variants, the switch is too long"
- Not for "the function returns void, so a missing case is fine"

## Why

Discriminated unions are only as useful as their consumers' coverage. A union with four variants and a `switch` that handles three is a bug surface, not a type. The compiler will not warn you — unless you make it.

The `never` assertion at the end of a switch turns "missing variant" into "build fails." When someone adds a fifth variant six months later, the compiler walks them straight to every consumer that needs updating.

This is one of the cheapest reliability wins in TypeScript: two lines, infinite future bugs prevented.

## When TS already enforces it

TypeScript narrows the discriminant inside each `case`. If **every case `return`s** and the function has a declared return type, TS will refuse to typecheck a missing case — the function would implicitly return `undefined` for the unhandled variant, and the return-type check catches it.

```ts
// ✅ No assertNever needed — TS rejects a missing case because the function
// must return string, and the missing variant produces an implicit undefined.
function label(status: 'idle' | 'loading' | 'success'): string {
  switch (status) {
    case 'idle':    return 'Ready';
    case 'loading': return 'Working…';
    case 'success': return 'Done';
  }
}
```

In this form, the Iron Rule is *already enforced by the type system*. Adding `default: assertNever(status)` is still a good habit:

- It produces a **clearer error message** when a variant is added (TS points to the `assertNever` line, not "function may not return").
- It adds a **runtime alarm** for the case where types lie (e.g., raw JSON parsed without a schema).
- It makes the intent explicit to readers who don't track which branches return.

But it's *not strictly required* in this form. The Iron Rule applies most strictly to the cases below, where TS *won't* save you.

## Detection

You are violating the rule if any of these are true (TS won't catch these without `assertNever`):

- A `switch (x.kind) { case 'a': ... case 'b': ... }` with **`void` return type** (e.g. event dispatch, side-effect functions) — TS can't use the return-type check.
- A `switch` where cases use `break` instead of `return`, with logic after the switch — TS narrows to `never` only at the case boundary, not after.
- A `switch` has `default: break;` or `default: return null;` over a closed set — the default *swallows* the missing case.
- A `switch` has `default: throw new Error('unreachable')` *without* the `never` check first — the comment is right but the type system isn't enforcing it.
- An `if/else` chain over `kind === 'a' | 'b' | 'c'` doesn't end in a typed exhaustiveness check.
- The phrase `// @ts-ignore — should never happen` appears anywhere near a switch.
- The discriminant is typed `string` or `number` (`switch (value: string)`), so `assertNever` *can't* compile — that loose type **is** the bug. Narrow the parameter to the closed union (`value: EventLocationType`) first; the exhaustiveness anchor only works once the type is closed.

## The Pattern

### The basic guard

```ts
export function assertNever(x: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(x)}`);
}
```

Two lines, ship once, use everywhere. When you'd rather not import a shared helper, the inline form is identical: `default: { const _exhaustive: never = event; throw new Error(`Unhandled: ${JSON.stringify(_exhaustive)}`); }` — same compile error when a variant is added. Reviewers should accept either form.

### Switch on a discriminated union

```ts
type Event =
  | { kind: 'created'; id: string }
  | { kind: 'updated'; id: string; patch: Patch }
  | { kind: 'deleted'; id: string };

function handle(event: Event): void {
  switch (event.kind) {
    case 'created':
      return persist(event);
    case 'updated':
      return applyPatch(event.id, event.patch);
    case 'deleted':
      return softDelete(event.id);
    default:
      return assertNever(event); // ← compile error if a variant is added without handling
  }
}
```

Now add a fourth variant:

```ts
type Event =
  | { kind: 'created'; id: string }
  | { kind: 'updated'; id: string; patch: Patch }
  | { kind: 'deleted'; id: string }
  | { kind: 'restored'; id: string }; // ← new
```

The compiler immediately flags `handle`:

```
Argument of type '{ kind: "restored"; id: string; }' is not assignable to parameter of type 'never'.
```

The build refuses to ship until you handle `'restored'`. The bug is prevented at compile time, not detected in production.

### Expression form (when the switch produces a value)

```ts
function label(status: 'idle' | 'loading' | 'success' | 'error'): string {
  switch (status) {
    case 'idle':    return 'Ready';
    case 'loading': return 'Working…';
    case 'success': return 'Done';
    case 'error':   return 'Failed';
    default:        assertNever(status);
  }
}
```

TypeScript narrows `status` to `never` at the `default`. The call typechecks. Add a variant; the build breaks.

### In rendering code

```ts
function FetchView({ state }: { state: FetchState<Order[]> }) {
  switch (state.status) {
    case 'idle':    return <EmptyState />;
    case 'loading': return <Skeleton />;
    case 'success': return <List items={state.data} />;
    case 'error':   return <ErrorBanner error={state.error} />;
    default:        return assertNever(state);
  }
}
```

JSX is just an expression. Same rule applies.

## Pressure Resistance

### "I'll never add a new variant, this is overkill"

You won't. The next maintainer will. They'll add the variant, they won't read every consumer, and the bug ships. The `never` check is a one-time cost that pays out every time the union evolves.

### "I'll use a `default` that throws — same thing, right?"

No. A `default: throw new Error('unreachable')` is a *runtime* check. The `never` assertion is a *compile-time* check. Runtime gives you a 500. Compile-time prevents the PR from merging.

### "The linter already catches missing cases"

Great — keep that rule on. But the `never` assertion still earns its place: linters run per-file at lint time; the type system runs at every consumer. Lint rules can be disabled per-line; the type system can't be silenced without `@ts-ignore`, which shows up in review.

### "The `assertNever` adds a runtime cost"

The cost is one function call in a path that, by construction, can't execute. If the path *does* execute, you have a bug, and you want to know.

### "My union has 20 variants, the switch is too long"

That's a different problem. Either the union is over-faceted (split into nested unions) or the consumer is doing too much (split the function). The exhaustiveness check is right; the abstraction below it is wrong.

## Red Flags

- A `switch` over a discriminant with no `default`, where the function returns `void`.
- A `default: break;` over a closed value set.
- The phrase "this should never happen" in any nearby comment.
- A `default: return null;` or `default: return undefined;` that the caller then has to defend against.
- An `if/else` chain over `kind === '...'` strings that doesn't end in `else { assertNever(...) }`.
- A code review comment "what about the new variant?" — you're already late.

**All of these mean: the exhaustiveness anchor is missing — add the `never` check.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "TypeScript already narrows, I don't need a runtime guard" | The runtime guard is the *anchor* that makes the compile-time narrowing strict. Without it, `default:` falls through quietly. |
| "I'll add the check later when the union grows" | "Later" is when the bug ships. The check costs nothing to add now. |
| "Throwing in production is worse than silent fallback" | Silent fallback is the bug. Throwing is the alarm. |
| "I want a generic default for forward-compat" | Forward-compat with what? You don't know what the new variant means; defaulting it is just guessing. |
| "The function returns `void`, so a missing case is fine" | The void return masks the bug. The data is unhandled and you'll never know. |
| "My switch is in a hot path, the function call is expensive" | A function call that runs zero times is free. If it's running, find the bug. |

## Related

- `make-illegal-states-unrepresentable` — the discriminated union the switch is exhaustive over
- `no-any-escape-via-unknown-or-never` — the `never` default guard; `any` would silence the check

## Reference

- Dan Vanderkam, *Effective TypeScript* 2e (2024), items 28–29 — union exhaustiveness as the canonical use of `never`.
- The TypeScript Handbook, ["Exhaustiveness checking"](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#exhaustiveness-checking) — official documentation of the pattern.
