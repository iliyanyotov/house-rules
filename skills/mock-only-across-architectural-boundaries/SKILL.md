---
name: mock-only-across-architectural-boundaries
description: Use when writing a test and reaching for `mock`, `spyOn`, or module-level mocks. Use when a test setup creates more than two mocks. Use when a test mocks a function defined in the same module as the unit-under-test. Use when a refactor that doesn't change observable behavior breaks the tests because internal mock interactions changed. Use when a test mocks a domain entity instead of just constructing one.
---

# Mock Only Across Architectural Boundaries

## Overview

**Test doubles replace things that cross an architectural boundary** — the network, the clock, the random source, third-party SDKs, sub-processes. They do *not* replace types your own code defines and owns. Those, you construct directly with real values.

A mock at the boundary is the testing analog of a port-and-adapter — it stands in for the world outside your code. A mock inside the boundary is a brittle simulation of code you could just *run*.

## The Iron Rule

```
NEVER mock code you own. Mock at architectural boundaries — the network, the clock, the SDK.
```

**No exceptions:**
- Not for "the helper is hard to set up"
- Not for "mocking is faster"
- Not for "London-school says mock everything"
- Not for "my test framework makes it easy"

## Why

Mocks are useful for one job: replacing a *non-deterministic*, *slow*, or *side-effecting* dependency with a deterministic, fast stand-in. The HTTP call to a payment provider. The DB write that costs 50ms. The `Math.random()` whose value you want to control. The clock you want to freeze.

Those are *real* benefits and the boundary is obvious — you don't own the payment provider; you don't want every test to spin up Postgres; you don't want a flaky test depending on system time.

Mocks become harmful when they cross *into* code you own. Three failure modes:

1. **They couple tests to implementation.** A mock asserts "the function called helper X with args Y." Refactor to inline X → test breaks even though behavior didn't change.
2. **They simulate something free to run.** Why mock `formatCurrency(100)` when you can just *call* `formatCurrency(100)`? The mock is a worse copy of the real thing.
3. **They hide design problems.** A test that needs to mock four owned types is testing a unit that imports four owned dependencies. The smell is in the unit, not the test.

The Khorikov framing: mockable boundaries are *unmanaged* dependencies — out-of-process, controlled by someone else. *Managed* dependencies (your DB, your code) you exercise live. Freeman & Pryce: mock at *adapter boundaries* (ports), not at internal collaborators. Same rule, two framings.

Two clarifications that often confuse:
- **The database is a managed dependency**, even though it's "out of process." You own its schema and migrations. Tests run against a real test DB (containers, branches, ephemeral instances) — *not* mocked.
- **The clock is an unmanaged dependency**, even though it's "in process." You don't own time. Inject a clock function and mock it in tests.

The boundary isn't strictly "in-process vs out-of-process." It's "code I own vs code I don't."

## Detection

You are violating the rule if any of these are true:

- A module-level mock on your own owned module.
- `spyOn` on a private helper in the same module being tested.
- A test constructs a mocked domain object (`mock<User>()`) to pass to the unit, when constructing a real `User` would work.
- A test mocks the entire query builder instead of running against a real test DB.
- A test's setup has 5+ mocks; the unit imports too much.
- A test asserts on a function call in the same module.
- A test mocks `Date.now()` directly via global stubbing instead of a clock seam.

## The Pattern

### Mock at the boundary; pass real values inside

```ts
// ❌ Mocks everything — owned User type, owned formatter, owned DB.
test('sendInvoice formats and emails', async () => {
  const mockUser = mock<User>({ email: 'a@b' });
  const mockFormatter = mock(() => 'formatted');
  const mockDb = { insert: mock(), select: mock() };
  const mockEmail = mock();

  await sendInvoice(mockUser, { format: mockFormatter, db: mockDb, email: mockEmail });

  expect(mockFormatter).toHaveBeenCalled();
  expect(mockDb.insert).toHaveBeenCalled();
});

// ✅ Real owned values; only the email send is mocked (the unmanaged boundary).
test('sendInvoice sends a formatted email to the user', async () => {
  const user = makeUser({ email: 'a@b' });          // real User
  const invoice = makeInvoice({ totalCents: 4200 }); // real Invoice
  const sentEmails: SendArgs[] = [];
  const email: SendFn = async (args) => { sentEmails.push(args); };

  await sendInvoice({ user, invoice }, { db, email });  // db is a real test DB

  expect(sentEmails).toHaveLength(1);
  expect(sentEmails[0].to).toBe('a@b');
  expect(sentEmails[0].html).toContain('$42.00');
});
```

The boundary is `email: SendFn` — an injected adapter at the edge of your code. The assertion is on the *observable* outcome (an email with the right payload). Real `User`, real `Invoice`, real `db`. Refactor any internal helper — the test stays green.

### Build domain entities with a schema-driven factory — don't mock them

Where do `makeUser` / `makeInvoice` come from? A factory builds a valid entity and merges your overrides — you specify only the fields the test cares about. Two common styles: **hand-curated** defaults (you write the base values once, in faker or plain literals) or **schema-generated** (a generator derives a valid instance from the schema). Hand-curated is the common case and needs no extra tooling; schema-generated auto-reflects schema changes if you have a generator.

```ts
// generateMock: produce a valid sample value from a schema (e.g. zod's generators,
// @anatine/zod-mock, or your own). Generically: "a valid instance of this schema."
const userFactory = (overrides: Partial<User> = {}): User => ({
  ...generateMock(userSchema),
  ...overrides,
});

// ✅ A real, fully-valid User. Every field satisfies userSchema.
const user = userFactory({ status: 'active' });
await suspendUser(user, { db: testDb });   // db is the real boundary, mocked/test-DB'd
```

Contrast with mocking the entity itself:

```ts
// ❌ Cast a partial object to the type — a fake User that drifts from userSchema.
const user = { id: '1', status: 'active' } as unknown as User;

// ❌ A library mock of the type — same problem, fancier syntax.
const user = mock<User>({ status: 'active' });
```

Both anti-patterns (a) **drift from the real schema** — the cast satisfies the compiler, not the validator, so an invalid `User` sails through; (b) **couple the test to fields it doesn't care about** — add a required field and you hand-edit every literal; (c) **hide shape changes** — when `User` gains or renames a field, the casts compile silently, while a factory updates in *one* place (and with a schema-generator, reflects the new shape automatically).

The split is the same rule as everywhere in this skill: **construct domain objects (via factories from the schema); reserve mocks for the boundaries** the unit talks to (DB / HTTP / external SDK). Factories keep test data valid-by-construction and refactor-safe; mocks stay at the seam.

### The clock — freeze it in the test, don't mock it

```ts
// ❌ Direct Date.now — tests fail at midnight in CI.
function isExpired(token: Token): boolean {
  return token.expiresAt.getTime() < Date.now();
}

// ✅ Production stays as-is; the test freezes the system clock.
//    freezeTime / restoreTime wrap whatever your runner provides for clock control.
afterAll(restoreTime);

test('token past its expiry reads as expired', () => {
  freezeTime(new Date('2026-01-15T00:00:00Z'));
  expect(isExpired(token)).toBe(true);
});
```

The clock is not an architectural boundary, so it doesn't need a mock *or* an injected seam — the test runner can pin `Date.now()` / `new Date()` directly. No global monkey-patching by hand, no flake, and production code carries no test-only parameter.

When you genuinely *can't* reach the runtime clock (a pure function you want frozen without touching global time, or a hot path where the global freeze is too broad), inject a `now` at the boundary instead:

```ts
function isExpired(token: Token, now: () => Date = () => new Date()): boolean {
  return token.expiresAt.getTime() < now().getTime();
}

expect(isExpired(token, () => new Date('2026-01-15T00:00:00Z'))).toBe(true);
```

That keeps `Date` at the *boundary* — the signature names a `now` dependency the test supplies. Reach for it only when freezing the global clock won't do; for most code, freezing it will.

### The DB is *not* mocked

```ts
// ❌ Mocking the query builder — tests pass; production breaks on column-name typos.
test('createOrder persists with status=pending', async () => {
  const mockInsert = mock(() => ({ returning: () => [{ id: 'order-1' }] }));
  await createOrder({ db: { insert: mockInsert } }, input);
  expect(mockInsert).toHaveBeenCalled();
});

// ✅ Real test DB (container / branch / in-memory).
test('createOrder persists with status=pending', async () => {
  const orderId = await createOrder({ db: testDb }, input);
  const persisted = await orders.findById(orderId, { db: testDb });
  expect(persisted.status).toBe('pending');
});
```

The mocked test gives false confidence — a typo in the column name, a missing index, a wrong table — the mock catches none of them. The DB test catches all of them because the boundary is exercised.

A two-tier split is the common production pattern: unit tests *may* mock the DB boundary with a **typed deep mock** of the client (so column/method typos are still caught by the types), while a separate integration suite — run on its own in CI — exercises the real schema. What's forbidden is letting a mocked DB be your *only* DB coverage: if you mock the client in a unit test, there must be an integration test that runs the real query.

Setup costs are real but bounded: spin up a test DB in `beforeAll`, reset between tests, drop in `afterAll`. Modern test runners keep this snappy.

### When a unit needs too many mocks — split the unit

```ts
// ❌ Single function imports four unmanaged deps — every test mocks four.
async function checkout(input: CheckoutInput) {
  await charge(input);           // payment provider
  await sendReceipt(input);      // email provider
  await notifyOps(input);        // ops webhook
  await trackEvent(input);       // analytics
}

// ✅ Split into operations; each has 1–2 mocks.
async function charge(input: CheckoutInput, deps: ChargeDeps) { /* ... */ }
async function sendReceipt(input: CheckoutInput, deps: EmailDeps) { /* ... */ }
async function notifyOps(input: CheckoutInput, deps: NotifyDeps) { /* ... */ }
async function trackEvent(input: CheckoutInput, deps: TrackDeps) { /* ... */ }

async function checkout(input: CheckoutInput, deps: AllDeps) {
  await charge(input, deps);
  await sendReceipt(input, deps);
  await notifyOps(input, deps);
  await trackEvent(input, deps);
}
```

Each leaf tests in isolation with 1–2 mocks. The orchestrator gets a small integration test that exercises real composition with mocked unmanaged deps.

### Quick reference

| Unmanaged (mock at the seam) | Managed (use the real thing) |
|---|---|
| External APIs (payment, email, LLM, search) | Your domain types — construct with builders |
| The clock (`now`, `Date.now`) | Your pure functions (formatters, validators, parsers) |
| Randomness (`random`, `randomUUID`) | Your DB queries — use a real test DB |
| File system, sub-processes | Your schemas — use the real schema |
| Browser globals in unit tests | Your handlers — test through the boundary with a real `Request` |
| Logging / error tracker SDKs | Your components — render and assert on DOM |

## Pressure Resistance

### "Mocking is faster than setting up a real DB"

For one test, yes. Across the whole suite, no — real DB tests catch real bugs, and the setup cost is paid once and spread across every test. Modern test runners with prepared fixtures keep DB tests under 100ms each. The "fast" mock test is also the test that lets a column-name typo ship.

### "Mocks isolate the unit-under-test"

Mocks isolate from *dependencies* — fine for unmanaged deps. They also isolate from *real collaborators* — bad for owned deps. The collaborator is *also* code you wrote and want to verify works together. Don't mock the thing you want to verify.

### "I have to mock the helper to test this branch"

Then the branch is in the wrong place — downstream of something you don't control from the unit's inputs. Restructure so the branch is reachable from the public input. Or: write an integration test that exercises the branch through the real path.

### "London-school says mock everything"

The London-school caricature does. The actual Freeman & Pryce position is *mock at ports* — adapters at architectural boundaries, not internal collaborators. The disagreement between London and classical schools is narrower than the caricatures suggest; both sides agree: don't mock owned types or pure functions.

### "My test framework makes mocking easy"

Tools enable; they don't recommend. Module-level mocking is a sharp tool. The rule isn't "no mocks" — it's "mocks at unmanaged boundaries, where they earn their weight."

## Red Flags

- A test uses a mocked domain type instead of constructing a real one.
- A test mocks a function defined in the same file as the unit-under-test.
- A test's `beforeEach` constructs 5+ mocks; the unit imports too much.
- A test asserts on internal helper call counts.
- A test mocks `Date.now()` via global stubbing instead of injecting a clock seam.
- A test mocks the query builder instead of using a real test DB.
- A refactor PR has 50+ test-fixture updates that don't correspond to behavior changes.
- A PR author defends a test with "but it mocks correctly" instead of "but it asserts the right behavior."

**All of these mean: the mock is inside the boundary — move it to the seam or remove it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Mocks are faster" | Faster per test, less safe per suite. The cost of false-pass mocks is paid in production. |
| "Mocks isolate the unit" | Mocks isolate from *dependencies*. Internal mocks also isolate from real collaborators you wanted to verify. |
| "London-school says mock everything" | London-school says mock at ports. Not the same. |
| "Setup is heavy without mocks" | Setup is once (test DB container in `beforeAll`); mock-overhead is every test, plus every test maintenance. |
| "Refactoring my test suite is a big task" | One test at a time; pair the refactor with feature work. Value compounds as the suite gets less mock-heavy. |
| "Tests pass; what's the problem?" | Tests passing against mocks but failing against real code is the *defining* problem of over-mocked suites. |

## Related

- `test-observable-behavior-not-implementation` — the corollary: assert outcomes, not internals
- `functional-core-imperative-shell` — mocks live only in the shell
- `seams-for-untestable-code` — seams give mocks a legitimate boundary to sit at

## Reference

- Vladimir Khorikov, *Unit Testing: Principles, Practices, and Patterns* (2020), ch. 5, ch. 8 — *"Use mocks only for unmanaged dependencies."* Distinguishes managed vs unmanaged.
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (2009) — "mock roles, not objects." Mock at ports (adapters), not at internal collaborators.
- Martin Fowler, ["Mocks Aren't Stubs"](https://martinfowler.com/articles/mocksArentStubs.html) (2007) — the original taxonomy. Classical-vs-London framing. Still the clearest treatment of when each style fits.
- Gary Bernhardt, "Boundaries" (2012) — functional-core / imperative-shell. A pure core has nothing to mock.
