---
name: composition-over-inheritance
description: Use when tempted to use class inheritance or `extends`. Use when subclass needs only some of the parent's behavior. Use when an inheritance hierarchy feels forced. Use when a base class accumulates methods that not all subclasses need. Use when reaching for "is-a" thinking before "has-a."
---

# Composition Over Inheritance

## Overview

**Favor composition over inheritance.** Build behavior by combining small, focused pieces — not by extending a parent class and overriding its methods.

Inheritance creates rigid hierarchies and tight coupling: every parent change ripples through every child, and you can only inherit from one parent. Composition creates flexible, swappable behavior: each piece is its own type, callers wire them together, and the same building block can be reused across unrelated contexts.

## The Iron Rule

```
NEVER use inheritance when composition would work. Default to composition; reach for inheritance only when a true subtype relationship exists.
```

**No exceptions:**
- Not for "it's the OOP way"
- Not for "is-a relationship"
- Not for "code reuse via extends"
- Not for "polymorphism"

## Why

Inheritance binds the child to the parent's *implementation* and *contract*, forever. A class that extends another inherits:

- Every public method (whether it makes sense for the child or not).
- The parent's internal field layout.
- The parent's lifecycle (constructor chain, hooks).
- The constraint that *one* parent must be enough.

When the parent grows, the child grows with it — including methods the child doesn't want. When the parent's contract narrows, every child must comply.

Composition is the opposite: the consumer says "I need this capability and that capability," wires them in, and that's the entire relationship. Each piece is a function or a small object with one job. Adding a third capability is adding a third dependency, not editing a hierarchy.

In a TypeScript/functional codebase, composition is the *natural* choice — function parameters, capability interfaces, and structural typing are all designed for it. Inheritance fights the language; composition fits it.

## Detection

You are violating the rule if any of these are true:

- A class declaration with `extends` for code reuse rather than a true subtype relationship.
- A subclass overrides a method to do nothing, throw an error, or change semantics.
- A base class accumulates methods that only some subclasses need.
- A change to a base class breaks half its children for reasons unrelated to the subtype relationship.
- "Manager"/"Service" / "Helper" / "Base" classes that exist primarily to hold shared methods.
- An `instanceof` check used to handle different subtypes at the call site — the abstraction isn't doing its job.
- A class hierarchy more than two levels deep (Grandparent → Parent → Child).

## The Pattern

### Compose capabilities, don't extend a hierarchy

```ts
// ❌ Inheritance with override-and-throw — classic LSP violation.
class Animal {
  eat(): void { /* ... */ }
  fly(): void { /* ... */ }
  swim(): void { /* ... */ }
}

class Penguin extends Animal {
  fly(): void {
    throw new Error("Penguins can't fly");
  }
}
```

```ts
// ✅ Capability interfaces. A class declares only what it can do.
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
  // No fly() — doesn't promise what it can't deliver.
}
```

The consumer types its dependency narrowly: a function that needs flying animals takes `Flies`. Penguins can't be passed in — and that's a compile error, not a runtime one.

### Function composition — the functional form

For most TS code, the simplest composition is *function parameters*:

```ts
// ❌ Inheritance for behavior reuse.
class BaseHandler {
  protected log(msg: string) { /* ... */ }
  protected authorize(req: Request) { /* ... */ }
}

class OrderHandler extends BaseHandler {
  async handle(req: Request) {
    this.log('handling order');
    this.authorize(req);
    // ... order-specific logic
  }
}
```

```ts
// ✅ Pass capabilities in as functions.
type Logger = (msg: string) => void;
type Authorizer = (req: Request) => void;

async function handleOrder(
  req: Request,
  deps: { log: Logger; authorize: Authorizer },
) {
  deps.log('handling order');
  deps.authorize(req);
  // ... order-specific logic
}
```

No hierarchy. No `this`-binding. Each capability is testable in isolation; each handler picks the dependencies it actually needs.

### When inheritance *is* the right tool

Inheritance earns its place when:

1. **There's a true `is-a` relationship** where the subtype genuinely can substitute for the supertype everywhere (see `liskov-substitution`).
2. **The hierarchy is shallow** (one level) and stable.
3. **The framework demands it** — e.g., extending an `Error` class for typed exceptions:

```ts
// ✅ Legitimate inheritance — true subtype, framework-required, one level deep.
export class InsufficientFundsError extends Error {
  constructor(readonly accountId: AccountId, readonly requested: number) {
    super(`Insufficient funds in ${accountId}: requested ${requested}`);
    this.name = 'InsufficientFundsError';
  }
}
```

If your inheritance doesn't fit those criteria, composition is the better tool.

### The shape-hierarchy trap

```ts
// ❌ The Square/Rectangle problem: inheritance via shape similarity.
class Rectangle {
  constructor(public width: number, public height: number) {}
  area(): number { return this.width * this.height; }
  setWidth(w: number): void { this.width = w; }
  setHeight(h: number): void { this.height = h; }
}

class Square extends Rectangle {
  // Square is "a kind of" Rectangle... or is it?
  setWidth(w: number): void { this.width = w; this.height = w; } // breaks Rectangle's contract
}
```

```ts
// ✅ Both implement a shared capability.
interface Shape {
  area(): number;
}

class Rectangle implements Shape {
  constructor(private width: number, private height: number) {}
  area(): number { return this.width * this.height; }
}

class Square implements Shape {
  constructor(private size: number) {}
  area(): number { return this.size * this.size; }
}
```

Square *has* an area; it isn't *a kind of* Rectangle. The hierarchy looked right because both have an area; the contract diverged on `setWidth`.

## Pressure Resistance

### "It's the standard OOP approach"

OOP-with-inheritance is one paradigm among several. Modern codebases — especially in TypeScript — lean on capability interfaces and function composition because they scale better. The fact that a textbook uses inheritance doesn't make inheritance the right tool for your code.

### "It's a true is-a relationship"

Maybe — but the Square/Rectangle problem shows that "is-a" intuition is unreliable. The test is *substitutability*: can the child replace the parent *everywhere* the parent appears, without breaking any caller? If not, it's not a true subtype.

### "Inheritance lets me reuse code"

Composition lets you reuse code with fewer constraints. A capability used by 10 unrelated types is one interface implemented 10 times; the same capability inherited from a base class means all 10 types share a parent that may eventually accumulate behavior they don't need.

### "Composition means lots of small pieces"

Yes — and that's the point. Small pieces are easy to name, test, and replace. Large inheritance hierarchies are hard to do any of those things with.

### "The framework requires extends"

When the framework genuinely requires it (extending `Error`, extending a framework-provided base class), use it. One level deep, with a clear `is-a` relationship, is the legitimate case. Don't generalize from there.

### "Composition is more code to write"

The composition version is *slightly* more code at the type-declaration site; it's *substantially* less code at every call site (because each call only depends on the capabilities it needs). Net cost is negative.

## Red Flags

- `extends` for reasons other than a true subtype relationship.
- A subclass override that throws, no-ops, or fundamentally changes the parent's contract.
- A base class with abstract methods that subclasses implement differently.
- An `instanceof` check used to dispatch on subtype — the abstraction failed.
- A hierarchy more than two levels deep.
- A "Base" or "Abstract" class whose only purpose is shared method storage.
- The phrase "let's just extend the existing class" without justification beyond "it has the methods we need."

**All of these mean: the inheritance is structural laziness — restructure as composition.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's the OOP way" | OOP has multiple ways. Composition is one of them, and the better default in TS. |
| "It's an is-a relationship" | Test substitutability. The intuition is unreliable. |
| "Inheritance reuses code" | Composition reuses code with fewer constraints. |
| "Composition is more files" | Each file is smaller, focused, and testable on its own. |
| "I only need one method from the parent" | Then take that one method as a parameter — don't inherit a whole class for it. |
| "Polymorphism requires inheritance" | TypeScript's structural typing gives you polymorphism without inheritance. |

## Reference

- *Design Patterns* (Gamma, Helm, Johnson, Vlissides — the "Gang of Four," 1994) — the original formulation: *"Favor object composition over class inheritance."*
- Sandi Metz, *Practical Object-Oriented Design* (2nd ed., 2018) — extended treatment of why inheritance is a constraint, not a tool, in most OO codebases.
- The TypeScript handbook's chapter on [structural typing](https://www.typescriptlang.org/docs/handbook/2/objects.html) — TS's type system makes capability-interface composition the natural style.
