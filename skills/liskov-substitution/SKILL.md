---
name: liskov-substitution
description: Use when creating a subtype (`extends` a class or `implements` an interface). Use when tempted to override a method with `throw` or a no-op. Use when an inheritance hierarchy or interface implementation feels forced. Use when handling subtypes via `instanceof` checks at the call site.
---

# Liskov Substitution

## Overview

**Subtypes must be substitutable for their base types without altering program correctness.** If `S` is a subtype of `T`, every place that takes a `T` should accept an `S` and still work. Subtypes must honor the contracts of their supertypes.

The "L" in SOLID. In a TypeScript codebase, it applies to both class inheritance (`extends`) *and* interface implementation (`implements`) — anywhere structural or nominal subtyping is in play.

## The Iron Rule

```
NEVER create a subtype that breaks the supertype's contract — no throw-in-override, no no-op-override, no semantic change.
```

**No exceptions:**
- Not for "it's the standard approach"
- Not for "I'll note it as an anti-pattern"
- Not for "the requirements say to extend it"
- Not for "throwing makes the limitation explicit"

## Why

The supertype is a *promise*: any caller holding a reference to `T` can rely on `T`'s methods behaving as documented. A subtype that violates that promise — by throwing, by no-op'ing, or by changing semantics — breaks every caller who didn't realize the reference might actually be the subtype.

The bug is invisible until it surfaces:
- A function takes `Bird` and calls `.fly()`. A `Penguin` extends `Bird` and overrides `.fly()` to throw. Every caller that previously assumed all birds fly now crashes when a `Penguin` is passed in.
- A function takes a `Writable` interface and calls `.write()`. A `ReadOnlyStorage` implements `Writable` with a no-op `.write()`. Data silently isn't persisted; the caller sees no error.
- A function takes `Rectangle`, sets width to 5 and height to 10, then asserts area is 50. A `Square` extends `Rectangle` and forces width == height. Area is 100, not 50. Caller's invariant breaks.

In each case, the supertype's contract was implicit; the subtype quietly violated it; the bug ships.

The fix is to **stop forcing inheritance/implementation that doesn't fit.** When a would-be subtype can't honor the contract, it isn't a subtype — it's a different type that happens to share *some* behavior. Use capability interfaces (small, focused) instead of broad inheritance.

## Detection

You are violating the rule if any of these are true:

- An overridden method throws an exception that the supertype's method doesn't.
- An overridden method is a no-op (silently does nothing).
- An overridden method changes the *semantics* of the supertype's method (different invariants, different side effects).
- An `instanceof` check appears at a call site to handle different subtypes differently *where the function expected them to behave uniformly* — the abstraction isn't doing its job. (Classifying a caught error by subtype at a boundary — `if (e instanceof UserError)` — is *not* this: the errors are being routed, not substituted into a uniform contract.)
- The supertype has a method that "doesn't apply" to a subtype.
- Adding a subtype requires updating every caller to handle the new case.

## The Pattern

### Don't subtype what you can't honor

```ts
// ❌ Subtype throws to "handle" what it can't do.
class Bird {
  fly(): void { /* ... */ }
  eat(): void { /* ... */ }
}

class Penguin extends Bird {
  fly(): void {
    throw new Error("Penguins can't fly"); // breaks every caller of fly()
  }
}
```

```ts
// ✅ Capability interfaces. Each type declares only what it can do.
interface Eats { eat(): void; }
interface Flies { fly(): void; }
interface Swims { swim(): void; }

class Sparrow implements Eats, Flies {
  eat(): void { /* ... */ }
  fly(): void { /* ... */ }
}

class Penguin implements Eats, Swims {
  eat(): void { /* ... */ }
  swim(): void { /* ... */ }
  // No fly() — doesn't claim it.
}

// Consumers type to the capability they need.
function takeFlight(flier: Flies) { flier.fly(); }

takeFlight(new Sparrow()); // ✅ compiles
takeFlight(new Penguin()); // ❌ compile error — Penguin doesn't implement Flies
```

The compile error is the point. A would-be runtime exception becomes a compile-time refusal.

### Don't change semantics in a subtype

```ts
// ❌ The classic Square/Rectangle violation.
class Rectangle {
  constructor(public width: number, public height: number) {}
  setWidth(w: number) { this.width = w; }
  setHeight(h: number) { this.height = h; }
  area(): number { return this.width * this.height; }
}

class Square extends Rectangle {
  // Square forces width = height. Breaks Rectangle's contract that
  // setWidth and setHeight are independent operations.
  setWidth(w: number) { this.width = w; this.height = w; }
  setHeight(h: number) { this.width = h; this.height = h; }
}

// A caller relying on Rectangle's contract:
function expandAndAssertArea(rect: Rectangle) {
  rect.setWidth(5);
  rect.setHeight(10);
  // Caller expects area 50. With a Square argument, area is 100.
}
```

```ts
// ✅ Both implement a shared capability — no false subtype relationship.
interface Shape {
  area(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  area(): number { return this.width * this.height; }
  setWidth(w: number) { this.width = w; }
  setHeight(h: number) { this.height = h; }
}

class Square implements Shape {
  constructor(private size: number) {}
  area(): number { return this.size * this.size; }
  setSize(s: number) { this.size = s; }
}
```

`Square` has an `area()`. It is not a kind of `Rectangle`.

### Don't no-op an override

```ts
// ❌ Silent failure: the read-only storage looks writable to a typed caller.
class FileStorage {
  read(path: string): string { /* ... */ }
  write(path: string, content: string): void { /* ... */ }
  delete(path: string): void { /* ... */ }
}

class ReadOnlyStorage extends FileStorage {
  write(path: string, content: string): void {
    // Silently does nothing. Caller writes, "succeeds," nothing is saved.
  }
  delete(path: string): void {
    // Same problem.
  }
}
```

```ts
// ✅ Split the capability surface; each implementation declares what it does.
interface Readable {
  read(path: string): string;
}

interface Writable {
  write(path: string, content: string): void;
  delete(path: string): void;
}

class FileStorage implements Readable, Writable {
  read(path: string): string { /* ... */ }
  write(path: string, content: string): void { /* ... */ }
  delete(path: string): void { /* ... */ }
}

class AuditLogStorage implements Readable {
  read(path: string): string { /* ... */ }
  // No write/delete — never claimed them.
}

// Function that needs to write declares it.
function persist(storage: Writable, path: string, content: string) {
  storage.write(path, content);
}
```

Passing `AuditLogStorage` to `persist` is now a compile error, not a silent runtime no-op.

### The compile-time substitution test

In TypeScript, the question "can `S` substitute for `T`?" is partly answered by the compiler — if `S` satisfies `T`'s structural type, the compiler will accept it. But the *behavioral* contract (semantics, invariants, error conditions) lives in your head and your tests. The compile-time check covers shape; the rule covers behavior.

Test it: write a small function that uses the supertype, and try every subtype. If any subtype breaks the function's assumptions, the subtype relationship is wrong — restructure to capability interfaces.

## Pressure Resistance

### "It's the standard approach to override-and-throw"

"Standard" in OO textbooks, maybe. In practice, override-and-throw produces runtime bugs that compile-time refusals would prevent. Move the limitation into the type system.

### "The requirements say Square extends Rectangle"

Requirements that mandate LSP violations are usually misunderstood. Push back: *"Square and Rectangle both have area, but Square's setWidth has different semantics. Two implementations of Shape is the safe model."* If the requirement genuinely forces the violation, document it as known tech debt and accept the cost.

### "Throwing in the override makes the limitation explicit"

A runtime throw is *less* explicit than a compile-time refusal. A caller reading the supertype's signature has no signal that this particular instance throws. The compile-time alternative (capability interfaces) makes the limitation visible to every caller, statically.

### "I'll note in the JSDoc that it might throw"

Documentation doesn't prevent the bug — it only explains it after the fact. The type system can prevent it before code runs.

### "It's just one method override"

The first one is. The next maintainer sees the pattern and adds a second. By the time the third lands, callers have no idea which methods are safe to call on which subtypes. Hold the line.

### "Composition would mean restructuring a lot"

Yes — when an LSP violation is deep in the codebase, the restructure is real. The cost of *not* restructuring is permanent: every caller now has to know which subtypes throw on which methods, which is a worse cost. Restructure once.

## Red Flags

- An overridden method that contains `throw new Error(...)` and the supertype's method doesn't.
- An overridden method whose body is empty or comment-only.
- An overridden method that calls `super.method()` and then "corrects" the result.
- An `instanceof Penguin` (or any subtype) check at a call site that takes the supertype *and expects uniform behavior from it*. (Routing a caught error by its subtype at an error boundary is legitimate, not a violation.)
- A subclass test that needs special setup the parent class's tests don't.
- A JSDoc comment "may throw if called on the X subtype."

**All of these mean: the subtype isn't a true subtype — split it into capability interfaces.** (LSP is about *substitution*, not about never inspecting type. Classifying a caught error by subtype to route it — user-facing 400 vs. re-thrown system error — is correct; capability interfaces are the wrong fix for error classification.)

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's the standard approach" | Common doesn't mean correct; it means commonly-buggy. |
| "I noted it in the docs" | Docs don't stop the bug; types do. |
| "The requirements force it" | Push back; if truly forced, document as debt. |
| "Throwing is explicit" | Compile errors are more explicit than runtime exceptions. |
| "No-op is safer than throw" | Silent failure is worse than loud failure. |
| "It's just for one method" | One violation invites more. Restructure now. |

## Related

- `composition-over-inheritance` — composition avoids the substitutability trap

## Reference

- Barbara Liskov, ["Data Abstraction and Hierarchy"](https://dl.acm.org/doi/10.1145/62139.62141) (1987) — the original "subtypes must be substitutable" formulation. The principle predates SOLID by a decade.
- Robert C. Martin, *Agile Software Development* (2002) — names LSP as the "L" in SOLID, and gives the canonical Square/Rectangle example.
- Joshua Bloch, *Effective Java* 3e (2018), Item 19 ("Design and document for inheritance or else prohibit it") — the modern restatement: most classes should be `final`; inheritance is a deliberate design choice, not a default.
