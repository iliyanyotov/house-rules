---
name: dependency-inversion
description: Use when a pure-logic file imports a network client, a database driver, a logger, an SDK, or any infrastructure module. Use when changing your DB provider or moving from REST to GraphQL would require editing the domain code. Use when business logic and infrastructure are intertwined in the same file. Use when designing the boundary between domain and adapters.
---

# Dependency Inversion

## Overview

**Inner code (domain, business logic, pure helpers) doesn't import outer code (database drivers, SDKs, frameworks).** Inner code declares the interface it needs; outer code provides the implementation.

Imports flow in one direction: **outer → inner, never the reverse.**

## The Iron Rule

```
NEVER let domain code import infrastructure. The domain declares the port; the adapter implements it.
```

**No exceptions:**
- Not for "it's so simple to just call the DB directly"
- Not for "we're not going to swap databases"
- Not for "ports are overengineered"
- Not for "I'd have to refactor everything"

## Why

When a pure function reaches up into infrastructure — a `domain/pricing.ts` file imports a DB client or `fetch` — the dependency is upside down. The fix is to declare the dependency *in* the domain (as a small interface or a function parameter) and let the imperative shell wire the real implementation in.

The original principle is the "D" in SOLID (Martin, 1996):

1. *High-level modules should not depend on low-level modules. Both should depend on abstractions.*
2. *Abstractions should not depend on details. Details should depend on abstractions.*

In a TS stack, that collapses to: **the domain layer never imports the infrastructure layer.** If `domain/pricing.ts` needs to look up a product, it doesn't `import { db }`; it takes a `getProduct: (id: ProductId) => Promise<Product>` parameter.

Why this pays off:

1. **The domain is portable.** Swap one DB library for another, Postgres for SQLite, REST for gRPC — the domain code doesn't change. Only the adapters change.
2. **Testing is trivial.** Unit-testing a domain function never requires mocking infrastructure — the test passes a fake adapter.
3. **The architecture's grain is visible.** Reading `domain/*` tells you the business rules. Reading `adapters/*` tells you the infrastructure. Mixing them muddles both.
4. **Cycles don't form.** When imports flow only outer-→inner, there's no path for a circular import.

## Detection

You are violating the rule if any of these are true:

- A file in `src/domain/`, `src/lib/`, or `src/utils/` imports from `src/db/`, `src/server/`, or any third-party SDK directly.
- A pure-logic function reaches `db.select()` or `fetch()` inside its body.
- A business-rule function takes infrastructure objects as parameters but doesn't accept the data it needs directly.
- Renaming or replacing an infrastructure module requires editing dozens of business-logic files.
- A unit test for a domain function requires a real DB, real network, or module-mocking.
- The same function appears in both a server context and a client context but can't be shared because it imports server-only infrastructure.

## The Pattern

### The domain declares the interface it needs

```ts
// ❌ Domain reaches into infrastructure.
// src/domain/pricing.ts
import { db, products } from '../infrastructure/db';

export async function calculateTotalPrice(orderItems: OrderItem[]): Promise<Money> {
  let total = Money.zero('USD');
  for (const item of orderItems) {
    const product = await db.select().from(products).where(eq(products.id, item.productId));
    total = total.add(product.price.scaleBy(item.quantity));
  }
  return total;
}
```

```ts
// ✅ Domain declares the port; doesn't know about the DB library.
// src/domain/pricing.ts
export type ProductReader = {
  findById: (id: ProductId) => Promise<Product | null>;
};

export async function calculateTotalPrice(
  reader: ProductReader,
  orderItems: OrderItem[],
): Promise<Money> {
  let total = Money.zero('USD');
  for (const item of orderItems) {
    const product = await reader.findById(item.productId);
    if (!product) throw new ProductNotFoundError(item.productId);
    total = total.add(product.price.scaleBy(item.quantity));
  }
  return total;
}
```

```ts
// src/adapters/dbProductReader.ts — NEW file, the adapter
import { db, products } from '../infrastructure/db';
import type { ProductReader } from '../domain/pricing';

export const dbProductReader: ProductReader = {
  async findById(id) {
    const [row] = await db.select().from(products).where(eq(products.id, id));
    return row ?? null;
  },
};
```

```ts
// src/handlers/orders.ts — the imperative shell wires them together
import { calculateTotalPrice } from '../domain/pricing';
import { dbProductReader } from '../adapters/dbProductReader';

export async function handlePostOrder(req: Request) {
  const items = await req.json();
  const total = await calculateTotalPrice(dbProductReader, items);
  return Response.json({ total: total.toJSON() });
}
```

**Read the imports:** `domain/pricing.ts` imports *nothing* from `infrastructure/`. The adapter imports from `domain/` (for the interface) and from `infrastructure/`. The handler imports both. **Imports flow inward; data flows out.**

### The "import direction" check

The single fastest way to verify is to look at imports. Pick a domain file:

```bash
# Should be empty or only ./domain/* imports.
grep -E "^import" src/domain/pricing.ts
```

If domain files import from `infrastructure/`, `db/`, third-party SDKs, or framework-specific modules, the dependency is upside down. The exception is *type-only* imports for shared shapes — those don't carry runtime dependency and are fine.

### Lighter form — the dependency is a function

For one-off cases where defining a `ProductReader` interface feels ceremonial, accept a function:

```ts
export async function calculateTotalPrice(
  findProduct: (id: ProductId) => Promise<Product | null>,
  orderItems: OrderItem[],
): Promise<Money> {
  /* ... */
}
```

This is dependency inversion in its leanest functional form. The function parameter *is* the abstraction. Composes naturally with the functional-core / imperative-shell split.

### Anti-Corruption Layer — for messy third parties

When a third-party API is messy (renamed fields, weird types, inconsistent error shapes), the adapter is also an *Anti-Corruption Layer* — it translates between the vendor's vocabulary and your domain's. Your domain never sees the vendor's quirks.

```ts
// Adapter translates the vendor's API shape into the domain's vocabulary.
export const vendorPaymentReader: PaymentReader = {
  async findById(id: PaymentId) {
    const charge = await vendor.charges.retrieve(id);
    return {
      id: PaymentId.parse(charge.id),
      amount: Money.of(charge.amount, Currency.parse(charge.currency.toUpperCase())),
      paidAt: new Date(charge.created * 1000), // vendor returns Unix seconds
      status: mapVendorStatus(charge.status),
    };
  },
};
```

The domain never sees the `charge.created`-in-seconds quirk; the adapter absorbs it.

### When this is overkill

The rule applies when **the domain has enough volume to deserve its own layer**. For a tiny project (≤30 files, ≤1 entity), splitting `domain/` and `adapters/` is overhead with no payoff. The signal that DIP is worth it: at least 5–10 domain functions sharing data sources, or confidence the infrastructure will change.

Don't DIP-ify speculatively. DIP-ify when the second adapter appears.

## Pressure Resistance

### "It's so simple to just call `db` directly"

Until the second test, the second consumer, or the second database arrives. Simple-to-write isn't the same as simple-to-maintain. The port + adapter is two extra files; it pays for itself by the third caller.

### "We're not going to swap databases"

Probably true. The portability argument is overrated for most products. The *testing* argument is the real win — a function with a port is testable in microseconds; a function that calls the DB directly requires module-mocking. The DB-swap scenario is the cherry on top.

### "Dependency injection is overkill"

We're not setting up a DI container. We're passing a function or an object literal as a parameter. The "DI" objection conflates Spring/Angular-style frameworks with the pattern of passing dependencies. The latter is a function parameter; that's all.

### "The domain ends up with too many interfaces"

If you find 15 narrow interfaces for 15 functions, group them. `ProductReader + ProductWriter` becomes `ProductRepository`. The granularity should match natural cohesion. The rule isn't "one interface per function"; it's "the domain doesn't import infrastructure."

### "I'd have to refactor everything"

You don't. Apply DIP to *new* code; leave existing code alone until you touch it. The codebase converges over months, not in one PR.

## Red Flags

- A file in `src/domain/`, `src/lib/`, or `src/utils/` with `import { db }`, an HTTP client, or a vendor SDK.
- A unit test that needs the real DB or module-mocking to run.
- A "service" or "manager" file mixing business logic and DB calls in one function body.
- A function whose name suggests pure computation (`calculateTotal`, `validateOrder`) that calls `await db.select()` inside.
- A "domain" type with fields named after DB columns (`created_at`, `org_id`) rather than domain terms (`createdAt`, `orgId`).
- Adapters that hold business logic (the adapter should be a thin translation layer).

**All of these mean: the import direction is inverted — declare the port in the domain, push the import into an adapter.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's just one DB call" | The first call sets the precedent. The second copies it. By call ten, the rule has been broken codebase-wide. |
| "Ports are overengineered" | A function parameter is a port. Overengineering would be a DI container; you're not building one. |
| "TypeScript already prevents bugs" | Types prevent some bugs. They don't prevent "we changed our DB provider and had to edit 80 files." |
| "We're a startup, no time" | Startups churn vendors more often than established codebases. DIP is *more* valuable at startup speed. |
| "Adapters add latency" | A function call adds nanoseconds. The actual DB call is milliseconds. The adapter's overhead is unmeasurable. |
| "The team won't follow the convention" | A code-review checklist plus a lint rule banning cross-layer imports enforces it mechanically. |

## Related

- `interface-segregation` — a narrow inverted port
- `composition-root-discipline` — DIP inverts imports; the composition root assembles the wiring
- `seams-for-untestable-code` — greenfield (DIP) vs. legacy-rescue (seams)

## Reference

- Robert C. Martin, *Agile Software Development* (2002) and *Clean Architecture* (2017) — the canonical "D" of SOLID. *Clean Architecture* generalizes it into "The Dependency Rule": source code dependencies point inward, toward higher-level policies.
- Alistair Cockburn, ["Hexagonal Architecture / Ports and Adapters"](https://alistair.cockburn.us/hexagonal-architecture/) (2005) — the pattern that operationalizes DIP at the application boundary. The domain hexagon doesn't depend on adapters; adapters implement the domain's ports.
- Eric Evans, *Domain-Driven Design* (2003) — the *Anti-Corruption Layer* pattern is DIP applied to third-party integrations.
