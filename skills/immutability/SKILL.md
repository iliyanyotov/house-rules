---
name: immutability
description: Use when designing a function parameter or a piece of shared state. Use when tempted to `.push`, `.splice`, `.sort` in place, reassign an object property, or pass an object that the callee will mutate. Use when reviewing a function that takes input and returns nothing but "modifies it for you."
---

# Immutability

## Overview

**Treat every value flowing into a function as `readonly`.** Return new values; never mutate inputs. Within a function, mutate only locally-constructed values that haven't escaped.

If a function changes the caller's data, the function's signature is lying.

## The Iron Rule

```
NEVER mutate a value the caller passed in. Build a new one and return it.
```

**No exceptions:**
- Not for "copying everything is slow"
- Not for "I'm the only caller, I know it's safe to mutate"
- Not for "sort in place is faster"
- Not for "JS arrays are passed by reference, mutation is idiomatic"

## Why

Mutation entangles call sites. A function that mutates its argument is invisible in the type signature: `function process(items: Item[]): void` looks identical whether it appends, sorts, or wipes the array. Callers have to *know* which one — and they don't.

Immutable inputs make functions composable. The same array can be passed to three functions without copy-defensiveness. Memoization works. Equality checks are meaningful. Time-travel debugging is feasible. UI frameworks that depend on reference equality (React, Solid) re-render correctly.

The cost is microscopic. Modern engines handle structural sharing well. The cost of mutation, on the other hand, is debugging shared-state bugs at 2am.

## Detection

You are violating the rule if any of these are true:

- A function takes `items: T[]` and calls `items.push`, `items.splice`, `items.sort`, `items.reverse`, or `items[i] = ...` inside.
- A function takes `obj: { ... }` and does `obj.field = newValue`.
- A reducer or state updater returns nothing and "modifies state in place."
- A test fixture is mutated between tests and the second test happens to pass.
- A function called `update`, `mutate`, `merge`, or `apply` doesn't return the new value.

## The Pattern

### Function parameters — readonly by default

```ts
// ❌ The signature doesn't say it mutates.
function sortByDate(orders: Order[]): void {
  orders.sort((a, b) => a.placedAt.getTime() - b.placedAt.getTime());
}

// ✅ Return a new array. Use the immutable native methods.
function sortedByDate(orders: readonly Order[]): readonly Order[] {
  return orders.toSorted((a, b) => a.placedAt.getTime() - b.placedAt.getTime());
}
```

Modern JS has `toSorted`, `toReversed`, `toSpliced`, `with` — all return new arrays without mutating. Use them.

### Updating objects — spread, don't assign

```ts
// ❌ Mutates the caller's object.
function markShipped(order: Order): Order {
  order.status = 'shipped';
  order.shippedAt = new Date();
  return order;
}

// ✅ New object, same shape, mutation isolated.
function markShipped(order: Order): Order {
  return { ...order, status: 'shipped', shippedAt: new Date() };
}
```

### Updating arrays — never index-assign

```ts
// ❌
function replaceAt<T>(arr: T[], i: number, v: T): T[] {
  arr[i] = v;
  return arr;
}

// ✅ — Array.prototype.with returns a new array.
function replacedAt<T>(arr: readonly T[], i: number, v: T): readonly T[] {
  return arr.with(i, v);
}

// ✅ — or via spread + slice for older targets.
function replacedAt<T>(arr: readonly T[], i: number, v: T): readonly T[] {
  return [...arr.slice(0, i), v, ...arr.slice(i + 1)];
}
```

### State updates — return new references

```ts
// ❌ The reference didn't change. Framework re-render won't fire.
const add = (item: Item) => {
  items.push(item);
  setItems(items);
};

// ✅ New array → new reference → re-render.
const add = (item: Item) => setItems(prev => [...prev, item]);
```

The same shape applies to any state container — reducers, stores, signals. Return a new value; never mutate the previous one.

### Reducers — return, don't mutate

```ts
// ❌ Mutates state in place.
const reducer = (state: State, action: Action): State => {
  state.count += 1;
  return state;
};

// ✅ Plain immutable reducer.
const reducer = (state: State, action: Action): State => ({
  ...state,
  count: state.count + 1,
});
```

The one defensible exception is a library that tracks "mutations" against a draft (like Immer's `produce`) — the draft is *not* the state; the library produces a new value at the end. Structurally still immutable.

### Marking it in the type — `readonly` and `Readonly<T>`

```ts
// Function signature: nothing inside can mutate.
function summarize(items: readonly LineItem[]): Summary { /* ... */ }

// Deep readonly for nested data.
type Frozen<T> = { readonly [K in keyof T]: T[K] extends object ? Frozen<T[K]> : T[K] };
```

`readonly` is type-level only — it prevents writes in TS but doesn't enforce at runtime. That's usually fine: the compiler catches violations during build.

### Query results are already immutable — don't fight it

```ts
// Query libraries return plain arrays. Don't mutate them.
const rows = await orders.findByOrgId(orgId);

// ❌
rows.sort(byDate);
return rows;

// ✅
return rows.toSorted(byDate);
```

## Pressure Resistance

### "Copying everything is slow"

In hot paths, measure. In application code, the cost is well below any threshold that matters. Modern engines structurally share; UI frameworks depend on reference equality for their own correctness — mutation defeats every optimization the framework offers.

### "I'm the only caller, I know it's safe to mutate"

You are *now*. The next person on the codebase is not. The signature lies; their code breaks. A function that doesn't mutate composes safely with every caller forever.

### "Immutable code is more verbose"

`{ ...obj, field: x }` is 18 characters. `obj.field = x` is 13. Five characters bought you a future without an entire class of bugs. Pay them.

### "Sort in place is faster"

`.sort()` mutates *and* runs in-place; `.toSorted()` does the same algorithm and allocates one new array. The cost difference is one allocation per call. If that allocation shows up in a flamegraph, you have a hot path; profile and decide locally. Until it does, prefer correctness.

### "But the function name is `updateUser`, mutation is implied"

Rename: `withUpdatedUser`, `updatedUser`, or use a verb-noun convention that returns the new value. Names that imply mutation lie about the type system; either fix the name or fix the implementation.

## Red Flags

- `.push`, `.pop`, `.splice`, `.sort`, `.reverse`, `.shift`, `.unshift` on an argument.
- `arr[i] = ...` for any non-locally-constructed array.
- `obj.field = ...` for any non-locally-constructed object.
- A function with return type `void` that takes a non-primitive argument and "does work."
- `Object.assign(input, { ... })` — the first argument is mutated.
- A test that runs in a different order than other tests "to avoid pollution."
- The phrase "the caller should make a copy" — no, the *callee* shouldn't mutate.

**All of these mean: the function's signature is lying — return a new value instead.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Performance" | Profile first. The cost is usually unmeasurable. |
| "JS arrays are passed by reference, mutation is idiomatic" | "Idiomatic" doesn't mean "correct." Modern array methods (`toSorted`, `with`, `toReversed`) exist because the idiom was wrong. |
| "I'm building it up incrementally" | Build it up in a *local* variable that hasn't escaped. Local mutation of locally-constructed values is fine. |
| "Spread copies are O(n)" | So is iterating to apply the change. The constant factor is identical. |
| "Mutation makes APIs simpler" | It makes the API *shorter*. It makes everything that consumes the API *more complex*. |

## Related

- `functional-core-imperative-shell` — pure cores are naturally immutable
- `small-changesets` — shared mutable state couples otherwise-separate changes

## Reference

- ECMAScript 2023, ["Change Array by Copy"](https://github.com/tc39/proposal-change-array-by-copy) — adds `toSorted`, `toReversed`, `toSpliced`, `with`. The language now provides immutable equivalents for every legacy mutating array method.
- Dan Vanderkam, *Effective TypeScript* 2e (2024), item 17 ("Use `readonly` to Avoid Errors Associated with Mutation") — the canonical TS treatment.
- Rich Hickey, ["The Value of Values"](https://www.infoq.com/presentations/Value-Values/) (JaxConf 2012) — the architectural case for immutability as the foundation of correctness.
