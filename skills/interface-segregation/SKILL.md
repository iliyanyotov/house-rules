---
name: interface-segregation
description: Use when typing a function parameter as a full SDK / framework / library object (Drizzle's `Database`, a payment SDK instance, an email SDK instance, `Request`) when the function only calls one or two of its methods. Use when writing a test fake and the mock object has to implement 30 methods because the parameter type demands it. Use when reviewing a function signature that takes a "do-everything" object.
---

# Interface segregation

## Overview

**A business-facing function's parameter type should describe the capability it uses**—not an entire upstream SDK. Prefer a hand-written port for stable domain contracts; use `Pick` when the selected upstream method has a simple, useful type.

A function that takes `db: Database` when it only calls `db.select` accepts more than it needs. The cost: tests must provide a full `Database` mock; refactors that change *any* method on `Database` ripple to this function's tests; the contract is unclear ("which methods does this function depend on?").

A function that takes `db: Pick<Database, 'select'>` accepts exactly what it needs. Tests provide `{ select: () => ... }` and nothing else; the contract is self-documenting; the rest of `Database` is free to evolve.

## The default rule

```
Prefer a narrow capability at business boundaries. Accept the full client only inside the
adapter/composition layer whose job is to own that client's lifecycle or coordinated surface.
```

Do not narrow mechanically. Fluent, generic, and overloaded SDK methods often make `Pick<Client, 'method'>` harder to fake and more coupled to vendor types than a small hand-written port.

## Why

The original ISP (Martin, 1996): *"Clients should not be forced to depend on methods they do not use."* The OO form is about not making implementers fulfill irrelevant interface methods; the TS form is about not making callers (especially tests) provide irrelevant fields.

The wins in a TS codebase:

1. **Tests get cheaper.** A `Pick<Database, 'select'>` parameter means tests pass `{ select: ... }` — not a 30-method mock.
2. **The contract is documented in the signature.** Reading the parameter type tells you exactly what the function touches.
3. **Refactors localize.** If `Database.insert`'s signature changes, only functions that *actually* call `.insert` are affected.
4. **Type-safe duck typing.** TS's structural typing makes this approach almost free at the type level — no nominal interface declarations, no implementation classes.

## Detection

You are violating the rule if any of these are true:

- A function parameter is typed as a full SDK class when only 1-2 methods are called.
- A test fake has to implement 10+ methods or fields it never uses (just to satisfy the parameter type).
- A function takes a full `Request` when it only reads `.headers.get('authorization')`.
- A "facade" type is shared across functions that use disjoint subsets of its surface.
- Test setup involves `as unknown as Database` or similar wholesale casts.
- `(arg as any).someProp` to read a property the wide framework type doesn't declare — the param is simultaneously *too wide* (carries 30 unused methods) and *too narrow* (lacks the field you actually need). A hand-written interface naming exactly the fields used both segregates and removes the `as any`.
- Adding a method to a popular type breaks tests for functions that don't use that method.
- The phrase "just take the full X, simpler" appears in code review.

## The pattern

### `Pick<T, ...>` — for simple upstream method types

```ts
type DirectoryUser = { id: string; email: string };
type DirectoryClient = {
  getUser(id: string): Promise<DirectoryUser | null>;
  searchUsers(query: string): Promise<DirectoryUser[]>;
  disconnect(): Promise<void>;
};

// ❌ Business function takes lifecycle/search capabilities it never uses.
export async function findOwnerEmailWide(client: DirectoryClient, id: string) {
  return (await client.getUser(id))?.email;
}

// ✅ The upstream method is simple, so Pick is an honest small contract.
export async function findOwnerEmail(
  client: Pick<DirectoryClient, 'getUser'>,
  id: string,
) {
  return (await client.getUser(id))?.email;
}

test('returns the owner email', async () => {
  const client = {
    getUser: async () => ({ id: 'u1', email: 'owner@example.com' }),
  };
  expect(await findOwnerEmail(client, 'u1')).toBe('owner@example.com');
});
```

`Pick` preserves the selected method's full type, overloads and generics included. Use it only when that extracted contract remains practical. Fluent query builders often do not; prefer a port such as `InvoiceReader.listByOrg` that returns the domain shape and test the adapter against the real database separately.

### Hand-written narrow interface

When the `Pick` is awkward (the underlying type is unwieldy, or you want to name the contract):

```ts
// ✅ Named narrow interface.
type InvoiceReader = {
  findById: (id: InvoiceId) => Promise<Invoice | null>;
  listByOrg: (orgId: OrgId) => Promise<Invoice[]>;
};

export async function generateMonthlyReport(reader: InvoiceReader, orgId: OrgId) {
  const invoices = await reader.listByOrg(orgId);
  /* ... */
}
```

Production wires it:

```ts
const invoiceReader: InvoiceReader = {
  findById: async (id) => (await db.query.invoices.findFirst({ where: eq(invoices.id, id) })) ?? null,
  listByOrg: (orgId) => db.query.invoices.findMany({ where: eq(invoices.orgId, orgId) }),
};
```

This is the "ports and adapters" pattern, with the port being the narrow interface.

### Narrow framework types

```ts
// ❌ Takes the whole Request just to read one header.
export function getAuthHeader(req: Request): string | null {
  return req.headers.get('authorization');
}
```

```ts
// ✅ Take only what's used.
export function getAuthHeader(
  req: { headers: { get: (name: string) => string | null } },
): string | null {
  return req.headers.get('authorization');
}
```

Now `getAuthHeader` is testable with any object that has `headers.get` — a plain `Headers` instance or `{ headers: { get: () => 'Bearer x' } }`. The `as unknown as Request` cast the wide signature forced (see Detection) is gone: the narrow param *is* the test fake's type, so no cast is needed.

### Namespaced clients (Prisma-style)

For `client.model.operation()` APIs, segregating to a model still drags in that model's full CRUD: `Pick<PrismaClient, 'creditBalance'>` narrows away the *other* models but keeps every `creditBalance` method. At a business boundary, hand-write the narrow shape:

```ts
type CreditBalanceReader = { creditBalance: Pick<PrismaClient['creditBalance'], 'findUnique'> };
```

Note that `Omit<PrismaClient, '$connect' | '$transaction' | …>` (the common `PrismaTransaction` idiom) is **not** interface-segregation — it strips ~6 lifecycle hooks but keeps every model, so a one-table repository method still receives the whole schema.

### Don't over-segregate

The rule is **type to what you use**, not **type to a single method per parameter**. If a function uses 4 methods of `Database`, the parameter type lists those 4. The cost of being too granular is too many narrow interfaces (each used by one function), which dilutes the benefit.

A useful heuristic: if your `Pick<>` lists 3+ method names and you have multiple functions doing similar work, extract a named interface (`InvoiceReader`, `Logger`, etc.). For 1-2 methods, inline `Pick` is fine.

### When the full client is the correct parameter

An adapter factory, transaction runner, migration tool, or composition root may intentionally coordinate the client's full lifecycle or several surfaces. Its contract *is* the client, so narrowing it invents a false abstraction. Keep the full type there; expose narrow domain ports from that layer to business code.

### Read-only vs read-write segregation

A common segregation: split the *read* and *write* halves of a dependency.

```ts
type InvoiceReader = {
  findById: (id: InvoiceId) => Promise<Invoice | null>;
};

type InvoiceWriter = {
  insert: (invoice: NewInvoice) => Promise<Invoice>;
  update: (id: InvoiceId, patch: Partial<Invoice>) => Promise<void>;
};

// A reporting function takes only the reader.
function report(reader: InvoiceReader) { /* ... */ }

// A migration function takes only the writer.
function backfillStatuses(writer: InvoiceWriter) { /* ... */ }

// A workflow that does both takes both.
function processOrder(reader: InvoiceReader, writer: InvoiceWriter) { /* ... */ }
```

This makes intent visible at the call site: "this function only reads" vs "this function writes" is encoded in the type.

## Pressure resistance

### "Pick is verbose"

For a simple method, an inline `Pick` is cheap. For a fluent or overloaded client, a named domain port is usually clearer and cheaper to fake. Judge the resulting contract, not its character count.

### "Tests already work fine with the full type"

Tests work *until* you change the upstream type. Then every test that mocked it has to gain or lose methods. Segregated parameter types localize the impact.

### "Hand-written interfaces are a maintenance burden"

Less than the alternative. The interface is a few lines; it's stable across SDK upgrades; it's a documented contract. The full type is dependent on the SDK author's choices and breaks tests on minor version bumps.

### "It's the same `db` instance — what's the point of pretending it's smaller?"

Two points: (1) the function signature documents what it *uses*, separately from what it *receives*. (2) Tests provide what the signature requires, not what production happens to have. The asymmetry is the win.

### "I'd have to import a new type"

For `Pick`, you import nothing new — it's built in. For hand-written interfaces, the type lives next to the function and costs one type declaration.

### "It's bad form to type to implementation details"

The point of segregation is the opposite — you're typing to the *minimum contract*, which is more abstract than the full library type. Less "implementation detail", not more.

## Red flags

- A test setup with `as unknown as` followed by a library type — almost always interface-segregation pressure.
- A function whose body uses 1-2 methods on a parameter whose type has 30.
- A "facade" or "service" type used by functions that each touch disjoint subsets.
- A bug report or PR where "this function broke because of an unrelated SDK upgrade."
- A mock fixture file with 100s of lines of unused method stubs.
- A function comment "only uses the X method" — the type should say that, not the comment.

**All of these mean: inspect the boundary. Business code usually needs a narrow port; an adapter that intentionally owns the full client may not.**

## Common rationalizations

| Excuse | Reality |
|---|---|
| "It's the same object at runtime" | Sure — but the parameter type is what tests and readers see. They're the audience. |
| "`Pick<...>` looks ugly" | Use it when the extracted method type is simple; otherwise name the domain capability with a hand-written port. |
| "I'd have to think about which methods to include" | That thinking *is* the design. It's the work. |
| "Tests just `as` the type" | `as` is the smell. Segregating avoids the smell altogether. |
| "We share types for consistency" | Share *narrow* types for genuine sharing. The full type is a category error. |
| "The framework provides the type — just use it" | Frameworks provide types as defaults. The point of TS's structural typing is you can be more specific. |

## Related

- `dependency-inversion` — DIP declares the port; ISP keeps it narrow
- `law-of-demeter` — both minimize what callers can reach (breadth vs. depth)

## Reference

- Robert C. Martin, *Agile Software Development: Principles, Patterns, and Practices* (2002) — the "I" in SOLID: "Clients should not be forced to depend on methods they do not use." (The oft-quoted "many client-specific interfaces are better than one general-purpose interface" is the common paraphrase.)
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (2009) — the "ports" pattern is interface-segregation applied at the *module* boundary: the domain code declares an interface for what it needs; adapters in the outer layer implement it.
