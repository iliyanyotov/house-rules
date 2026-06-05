---
name: deep-modules
description: Use when designing a new module, package, or function. Use when tempted to expose every internal helper, write thin pass-through wrappers, or add another configuration option "for flexibility." Use when reviewing a class with 30 public methods or a function with 12 parameters. Use when a feature requires touching seven files to make a small change.
---

# Deep Modules

## Overview

**A module's interface should be much simpler than its implementation.** Hide complexity behind small, expressive surfaces. Never expose internals merely because they exist.

The metric: interface size vs. implementation size. Small interface + large implementation = deep (good). Large interface + small implementation = shallow (bad).

## The Iron Rule

```
NEVER expose more surface than callers need. The interface is the contract; everything else is private.
```

**No exceptions:**
- Not for "flexibility — more options means callers can do more"
- Not for "I'll expose it just in case someone needs it"
- Not for "it's just a quick utility, no need to design it"
- Not for "hiding things makes debugging harder"

## Why

Every public name is a contract you must maintain forever. Every parameter is a decision the caller must make. Every option is a combinatorial explosion of test cases and edge interactions.

Shallow modules look harmless — "just a thin wrapper" — but the cost of *using* them is high: callers wire together five things instead of one, and the underlying complexity leaks through. Deep modules absorb the complexity. The caller sees one verb.

A 200-line function that does one important job behind one well-named entry point beats five 40-line functions whose composition is the caller's problem.

## Detection

You are violating the rule if any of these are true:

- A function passes through ≥4 parameters unchanged to another function.
- A module exports more than ~5 public names where most are used together.
- A class has getters/setters for every field — that's an object, not a module.
- An "options bag" has 8+ fields the caller almost always copies from a previous call.
- A package's README has 30 named functions and you can't tell which one to start with.
- Two callers always call `A`, `B`, `C` in the same order — that's not three functions, that's one.
- A change to internal behavior requires updating five public signatures.

## The Pattern

### Repository — deep, not shallow

```ts
// ❌ Shallow. Caller composes the work — has to know about transactions, ordering, validation.
export const usersRepo = {
  insert: (data: NewUser) => db.insertUser(data),
  setEmail: (id: UserId, email: string) => db.updateUser(id, { email }),
  setStatus: (id: UserId, status: Status) => db.updateUser(id, { status }),
  withTx: (fn: (tx: Tx) => Promise<void>) => db.transaction(fn),
  validate: (data: NewUser) => userSchema.parse(data),
};

// Every caller writes:
//   await usersRepo.withTx(async tx => {
//     usersRepo.validate(input);
//     const user = await tx.insertUser(...);
//     await tx.updateUser(...);
//   });

// ✅ Deep. One named operation per use case.
export const usersRepo = {
  /** Insert a new user, validate input, return the created row. */
  async create(input: NewUserInput): Promise<User> {
    const data = NewUser.parse(input);
    return db.insertUser(data);
  },

  /** Update the user's email after parsing the new value. */
  async changeEmail(id: UserId, email: string): Promise<void> {
    const valid = z.string().email().parse(email);
    await db.updateUser(id, { email: valid });
  },
};

// Caller writes:
//   await usersRepo.create(input);
```

The deep version is more code internally — and far less code at each call site. Three call sites? Six lines saved. Thirty call sites? Sixty lines saved.

### Function options — collapse what callers always co-set

```ts
// ❌ Twelve options the caller has to think about.
function fetchOrders({
  orgId,
  status,
  startDate,
  endDate,
  limit,
  offset,
  sortBy,
  sortDir,
  includeArchived,
  withCustomer,
  withLineItems,
  timezone,
}: Options) { /* ... */ }

// ✅ Three named use-cases that absorb the options.
async function listOrdersForOrg(orgId: OrgId): Promise<Order[]> { /* ... */ }
async function listOpenOrdersForOrg(orgId: OrgId): Promise<Order[]> { /* ... */ }
async function exportOrdersForOrg(orgId: OrgId, range: DateRange): Promise<Order[]> { /* ... */ }
```

The combinatorial options didn't disappear — they're internal to each function, configured once for the use case. The *caller* writes one name per intent.

### Pass-through wrapper — usually wrong

```ts
// ❌ Adds a layer with no decisions of its own.
export function sendEmail(opts: EmailOptions) {
  return mailer.send(opts);
}

// ✅ If the wrapper exists, it should hide complexity — templates, retry, logging.
export async function sendWelcomeEmail(user: User): Promise<void> {
  await mailer.send({
    from: WELCOME_SENDER,
    to: user.email,
    subject: 'Welcome',
    html: renderWelcomeTemplate(user),
    headers: { 'X-Entity-Ref-ID': user.id },
  });
}
```

The first version is `mailer.send` with extra typing — a tax with no benefit. The second hides the template, the sender, the headers, and adds testable intent.

### Ousterhout's example — file I/O

The classic shallow design is C's `read`, `write`, `lseek`, `open`, `close` — five primitives, the caller assembles. The deep design is Python's `open(path).read()` — two methods, one returns the entire file. The Python interface hides the seek/buffer/state machine. The C interface exposes it. Same problem, two depths.

## Pressure Resistance

### "Flexibility is good — more options means callers can do more"

Flexibility moves complexity from the *module* to *every caller*. Ten callers with custom configurations is ten subtly different uses to maintain. One named operation with internal sensible defaults is one. Add options only when *current callers actually disagree*; otherwise, defer.

### "Hiding things makes debugging harder"

Hiding *implementation* doesn't hide behavior. The caller still sees inputs and outputs; the trace still shows the path. What's hidden is *combinatorial complexity* — and that complexity wasn't helping anyone debug; it was creating new bugs.

### "I'll expose it just in case someone needs it"

YAGNI applies. An exposed name is harder to remove than to add. Wait for the second real consumer before generalizing.

### "Deep modules are harder to test"

The opposite. A deep module has *one entry point per use case* — each is tested with that use case's inputs and expected outputs. Shallow modules require tests of every primitive *and* of the compositions, which multiplies the surface.

### "Single-responsibility says split everything"

Single-responsibility says **one reason to change**, not **small surface**. A deep module has one reason to change (its use case) and a large implementation. That's compatible.

### "It's just a quick utility, no need to design it"

Utilities are the worst offenders. They grow public-by-accident: someone exposes a helper "just for this test," then it's in the API forever. Start utilities private; promote to public only when justified.

## Red Flags

- A module's `index.ts` re-exports everything in the directory.
- A function with `Options` or `Config` parameter containing 8+ fields.
- A wrapper function whose body is `return underlying(...args)` plus minor renaming.
- A class with public method counts in the double digits.
- A README that documents 20 separate functions with no "start here" entry.
- A code review that asks "should I import the helper or build it inline" — that's a sign the helper isn't load-bearing enough to exist.
- The phrase "let me expose this so the test can poke at internals" — restructure the test, not the module.

**All of these mean: the interface is too wide — collapse to named use cases.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Exposing the building blocks is more composable" | Composability happens *inside* the deep module. Externally, callers want named operations. |
| "Generic functions are reusable" | Generic functions are *unusable* without ten callers each writing the same wrapper. |
| "Adding options is forward-compatible" | Adding options is backward-incompatible the moment someone passes a non-default. |
| "It's only a wrapper, the cost is zero" | Wrappers without decisions of their own are tax. The cost is reading them and not learning anything. |
| "What if I need to swap implementations later" | When you need it, the deep interface is exactly what makes the swap possible. The narrow port is the swap-friendly shape. |
| "Fluent chains read nicely" | Fine when the chain has internal state. Useless when the chain is just an alternate syntax for an options bag. |

## Reference

- John Ousterhout, *A Philosophy of Software Design* (2018), ch. 4 ("Modules Should Be Deep") — the original framing of *information hiding* and *interface vs. implementation cost*. The substring and file-I/O examples are his.
- David Parnas, ["On the Criteria to Be Used in Decomposing Systems into Modules"](https://www.win.tue.nl/~wstomv/edu/2ip30/references/criteria_for_modularization.pdf) (1972) — the 50-year-old foundation: hide design decisions inside modules so they can change without breaking callers.
