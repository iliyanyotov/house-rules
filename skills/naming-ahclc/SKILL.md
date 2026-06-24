---
name: naming-ahclc
description: Use when naming a function, hook, file, variable, or symbol. Use when reaching for `handleStuff`, `getData`, `doProcess`, `manager`, `helper`, or `utils`. Use when a boolean isn't prefixed with `is` / `has` / `should`. Use when a collection variable is singular or a single-value variable is plural. Use when reviewing a name that requires its surrounding code to make sense.
---

# Naming: A/HC/LC

## Overview

**A name must say what the symbol *does* and *what it returns* — without reading its body.**

Names are the most-read part of any codebase. A good name eliminates the need to read the implementation; a bad name forces re-parsing on every encounter.

## The Iron Rule

```
NEVER use a name that needs its surrounding code to be understood.
```

If you have to read the body to know what the function does, the name is wrong.

## The Pattern: Action + High Context + Low Context

Read left-to-right, the name answers: *what does it do, on what, in which scope?*

- `getUser` — Action: `get`. HC: `User`. LC: none.
- `getUserMessages` — Action: `get`. HC: `User`. LC: `Messages`.
- `getUserUnreadMessages` — Action: `get`. HC: `User`. LC: `UnreadMessages`.
- `markInvoiceAsPaid` — Action: `mark`. HC: `Invoice`. LC: `Paid`.

### Action verbs — pick from a fixed set, don't invent

| Verb | Use for |
|---|---|
| `get` | Pure or memoized retrieval against a long-lived resource |
| `fetch` | Per-call network or DB retrieval |
| `load` | Bootstrap retrieval at app start |
| `set` | Assignment to a known target |
| `compute` | Derive a value from inputs (pure) |
| `compose` | Build a value from parts |
| `parse` | Convert `unknown` into a typed value |
| `format` | Convert a typed value into a display string |
| `mark` | Set a status flag on a domain entity |
| `apply` | Map a transformation over data |
| `handle` | Dispatch an event or action (at boundaries) |
| `assert` | Throw if a condition fails |
| `ensure` | Idempotently establish a precondition |

**Avoid:** `do`, `process`, `manage`, `perform`, `execute`, `run` — they don't tell you *what*.

**The `get` vs `fetch` carve-out.** The strict reading (`get` = sync, `fetch` = async) collides with established convention — `getCurrentUser`, `getSession`, `getDb` are all async and idiomatically named `get*`. The practical rule: **`get*` for memoized lookups** against a long-lived resource (the cost is paid once and reused, conceptually a lookup), **`fetch*` for per-call network operations** that hit the wire every time.

### Boolean prefixes — mandatory

| Prefix | Use for |
|---|---|
| `is` | Stateful predicate — `isLoading`, `isAdmin`, `isExpired` |
| `has` | Possession — `hasAccess`, `hasPaid`, `hasError` |
| `should` | Recommendation — `shouldRetry`, `shouldRender` |
| `can` | Capability — `canEdit`, `canRefund` |

No prefix → not a boolean. `loading: boolean` is wrong; either rename to `isLoading` or model the state as a discriminated union.

### Cardinality — singular for scalars, plural for collections

```ts
// ❌ Lies about cardinality.
const user = await db.users.findMany();    // returns an array, named singular
const items = list.find(p => p.id === id); // returns one item, named plural

// ✅ Match the shape.
const users = await db.users.findMany();
const user = await db.users.findUnique({ where: { id } });
```

A function returning a collection ends in plural (`getInvoices`); a function returning a scalar uses singular (`getInvoice`).

### Context — enough to disambiguate, no more

```ts
// ❌ Over-named — duplicates surrounding context.
class UserService {
  getUserById(userId: string) { /* ... */ }   // "User" said twice
}

// ✅ The class IS the context.
class UserService {
  getById(id: UserId) { /* ... */ }
}

// ❌ Under-named — generic helper without an anchor.
function format(x: Date): string { /* ... */ }

// ✅ Names what it formats.
function formatInvoiceDate(d: Date): string { /* ... */ }
```

## Worked Examples

```ts
// Pure derivations — compute / format
function computeOrderTotal(items: readonly LineItem[]): number { /* ... */ }
function formatPriceCents(cents: number): string { /* ... */ }

// Async data — fetch
async function fetchInvoiceById(id: InvoiceId): Promise<Invoice | null> { /* ... */ }
async function fetchOverdueInvoices(orgId: OrgId): Promise<Invoice[]> { /* ... */ }

// Memoized lookup — get
async function getCurrentUser(): Promise<User | null> { /* ... */ }

// Side effects — mark / send / apply
async function markInvoiceAsPaid(id: InvoiceId, at: Date): Promise<void> { /* ... */ }
async function sendWelcomeEmail(user: User): Promise<void> { /* ... */ }

// Booleans — is / has / should / can
function isInvoiceOverdue(invoice: Invoice, now: Date): boolean { /* ... */ }
function hasPaymentMethod(user: User): boolean { /* ... */ }
function shouldRetry(error: Error, attempt: number): boolean { /* ... */ }

// Parsers — parse
function parseEnv(raw: unknown): Env { /* ... */ }

// Event handlers — handle
function handleSubmit(e: FormEvent) { /* ... */ }
```

## Pressure Resistance

**"The context makes it clear."** Context decays. The PR author has full context; the reviewer has half; the bug-fixer six months later has none. Name for the bug-fixer.

**"Renaming touches 80 files."** Then it's wrong in 80 files. Modern IDE refactor tools rename safely; the cost only goes up over time.

**"Long names are ugly."** Wrong is uglier. `markInvoiceAsPaid` is two more words than `update` and infinitely clearer at the call site. Editors autocomplete; you type the long name once.

**"`get` and `fetch` are interchangeable."** In casual JS, yes. In a disciplined codebase, the distinction tells the reader whether to expect a per-call network hit. Reserving the verbs is cheap; conflating them is permanent ambiguity.

**"The framework uses `handle` without qualifiers."** Framework conventions override at framework boundaries — `<form onSubmit={handleSubmit}>` is fine because the JSX provides the LC. A `handleSubmit` *exported from a module*, on the other hand, is too vague.

## Common Mistakes

| Smell | Fix |
|---|---|
| `data`, `result`, `value`, `item`, `obj`, `info`, `thing` as standalone names | Name the *kind* of data |
| Files named `utils.ts`, `helpers.ts`, `common.ts`, `misc.ts`, `lib.ts` | Split by domain; name by content |
| Function name contains `And` or `Or` | Split — it has multiple responsibilities |
| Class suffixed `Manager`, `Helper`, `Util`, `Coordinator` (says it does *something*, not what) | Name the actual thing it does. `*Service`/`*Repository` are accepted layer names *with a domain-noun prefix* (`PaymentService` ✓); a do-something prefix is still a smell (`SMSManager` → `SMSSender`) |
| Boolean without an `is` / `has` / `should` / `can` prefix | Add the prefix, or model as a discriminated union |
| Returned collection named in singular | Pluralize, or unwrap to a scalar |
| Abbreviations: `btn`, `usr`, `cfg`, `mgr`, `svc` | Spell it out |
| `get*` that hits the wire on *every* call (no memoization) | Rename to `fetch*`. Memoized/amortized lookups (`getCurrentUser`, `getSession`) stay `get*` |

## The Bottom Line

**A name is a contract with the next reader. Make it complete.**

Action + High Context + Low Context. Booleans get prefixes. Collections are plural. If the name needs context to make sense, the name is wrong.

## Related

- `comments-as-why-not-what` — good names eliminate "what" comments
- `ubiquitous-language` — names must use the domain's words
- `principle-of-least-astonishment` — a name that predicts behavior

## Reference

- A/HC/LC pattern — [`kettanaito/naming-cheatsheet`](https://github.com/kettanaito/naming-cheatsheet). The canonical source for the convention.
- Robert C. Martin, *Clean Code* (2008), ch. 2 ("Meaningful Names") — the foundational treatment of intention-revealing names.
- Tim Ottinger, "Ottinger's Rules for Variable and Class Naming" — "use intention-revealing names; avoid disinformation."
