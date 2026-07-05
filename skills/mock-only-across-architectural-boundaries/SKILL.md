---
name: mock-only-across-architectural-boundaries
description: Use when writing a test and reaching for `mock`, `spyOn`, or module-level mocks. Use when a test setup creates more than two mocks. Use when a test mocks a function defined in the same module as the unit-under-test. Use when a refactor that doesn't change observable behavior breaks the tests because internal mock interactions changed. Use when a test mocks a domain entity instead of just constructing one.
---

# Mock Only Across Architectural Boundaries

## Overview

**Test doubles replace observable ports and nondeterministic boundaries**—network providers, clocks, random sources, subprocesses, and narrow repository ports in unit tests. They do *not* replace internal implementation details or domain values. Construct domain values directly and verify managed adapters with integration tests.

A mock at a port is the testing analog of an adapter—it supplies a controlled collaborator through an explicit contract. Mocking an internal helper is a brittle simulation of code the test could simply run.

## The Iron Rule

```
NEVER mock internal implementation details or domain values. Double explicit ports and
nondeterministic boundaries; integration-test managed adapters against the real dependency.
```

**No exceptions:**
- Not for "the helper is hard to set up"
- Not for "mocking is faster"
- Not for "London-school says mock everything"
- Not for "my test framework makes it easy"

## Why

Mocks are useful for one job: replacing a *non-deterministic*, *slow*, or *side-effecting* dependency with a deterministic, fast stand-in. The HTTP call to a payment provider. The DB write that costs 50ms. The `Math.random()` whose value you want to control. The clock you want to freeze.

Those are *real* benefits: you do not want every unit test to call a payment provider, spin up Postgres, or depend on wall-clock time. Integration tests separately exercise the managed adapters you need confidence in.

Mocks become harmful when they replace internal implementation rather than an explicit port. Three failure modes:

1. **They couple tests to implementation.** A mock asserts "the function called helper X with args Y." Refactor to inline X → test breaks even though behavior didn't change.
2. **They simulate something free to run.** Why mock `formatCurrency(100)` when you can just *call* `formatCurrency(100)`? The mock is a worse copy of the real thing.
3. **They hide design problems.** A test that needs four unrelated port doubles may be testing a unit with too many responsibilities. The smell can be in the unit, not the doubles.

The useful axis is **architectural role**, not ownership. A port interface may be code you own and still be the correct unit-test seam; a private helper is code you own and is not. Managed dependencies such as your database require real integration coverage even when a unit test uses a narrow repository double.

Two clarifications that often confuse:
- **The database is a managed dependency.** Adapter/integration tests run against a real test DB. Pure domain unit tests may double a narrow repository port; they do not mock a fluent query builder and call that database coverage.
- **The clock is a nondeterministic boundary.** Freeze it via the test runner's clock control (preferred), or inject a `now` seam where global control is too broad—but never hand-monkey-patch `Date.now`.

The boundary is not strictly in-process/out-of-process or owned/unowned. Ask whether the collaborator is an explicit observable port, nondeterministic source, or internal implementation detail.

## Detection

You are violating the rule if any of these are true:

- A module-level mock on an internal helper/module rather than an explicit port.
- `spyOn` on a private helper in the same module being tested.
- A test constructs a mocked domain object (`mock<User>()`) to pass to the unit, when constructing a real `User` would work.
- A mocked query builder is the only coverage for database behavior.
- A test's setup has 5+ mocks; the unit imports too much.
- A test asserts on a function call in the same module.
- A test hand-monkey-patches `Date.now` (e.g. `Date.now = () => ...`) instead of using the runner's clock control or a `now` seam.

## The Pattern

### Mock at the boundary; pass real values inside

```ts
// ❌ Mocks domain values, pure formatting, and internal call choreography.
test('sendInvoice formats and emails', async () => {
  const mockUser = mock<User>({ email: 'a@b' });
  const mockFormatter = mock(() => 'formatted');
  const mockDb = { insert: mock(), select: mock() };
  const mockEmail = mock();

  await sendInvoice(mockUser, { format: mockFormatter, db: mockDb, email: mockEmail });

  expect(mockFormatter).toHaveBeenCalled();
  expect(mockDb.insert).toHaveBeenCalled();
});

// ✅ Real domain values; the email port is doubled and the DB adapter is exercised live.
test('sendInvoice sends a formatted email to the user', async () => {
  const user = makeUser({ email: 'a@b' });          // real User
  const invoice = makeInvoice({ totalCents: 4200 }); // real Invoice
  const sentEmails: SendArgs[] = [];
  const email: SendFn = async (args) => { sentEmails.push(args); };

  await sendInvoice({ user, invoice }, { db, email });  // db is a real test DB

  expect(sentEmails).toHaveLength(1);
  expect(sentEmails[0]?.to).toBe('a@b');
  expect(sentEmails[0]?.html).toContain('$42.00');
});
```

The boundary is `email: SendFn` — an injected adapter at the edge of your code. The assertion is on the *observable* outcome (an email with the right payload). Real `User`, real `Invoice`, real `db`. Refactor any internal helper — the test stays green.

### Build domain entities with a schema-driven factory — don't mock them

Where do `makeUser` / `makeInvoice` come from? A factory builds a valid entity and merges your overrides — you specify only the fields the test cares about. Two common styles: **hand-curated** defaults (you write the base values once, in faker or plain literals) or **schema-generated** (a generator derives a valid instance from the schema). Hand-curated is the common case and needs no extra tooling; schema-generated auto-reflects schema changes if you have a generator.

```ts
// generateMock: produce a valid sample value from a schema (e.g. zod's generators,
// @anatine/zod-mock, or your own). Generically: "a valid instance of this schema."
const userFactory = (overrides: Partial<User> = {}): User =>
  userSchema.parse({
    ...generateMock(userSchema),
    ...overrides,
  });

// ✅ A real, fully-valid User. Every field satisfies userSchema.
const user = userFactory({ status: 'active' });
await suspendUser(user, { db: testDb });   // adapter behavior uses the real test DB
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

The clock is a nondeterministic boundary, but it does not always need a constructor seam: the test runner can pin `Date.now()` / `new Date()` directly. No global monkey-patching by hand, no flake, and production code carries no test-only parameter.

When you genuinely *can't* reach the runtime clock (a pure function you want frozen without touching global time, or a hot path where the global freeze is too broad), inject a `now` at the boundary instead:

```ts
function isExpired(token: Token, now: () => Date = () => new Date()): boolean {
  return token.expiresAt.getTime() < now().getTime();
}

expect(isExpired(token, () => new Date('2026-01-15T00:00:00Z'))).toBe(true);
```

That keeps `Date` at the *boundary* — the signature names a `now` dependency the test supplies. Reach for it only when freezing the global clock won't do; for most code, freezing it will.

### The DB adapter is tested live

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

A two-tier split is the common production pattern: domain unit tests double a **narrow repository port**, while a separate integration suite exercises the real adapter, query, migrations, and schema. Avoid typed deep mocks of fluent database clients: they couple tests to query-builder choreography while still missing database behavior. What's forbidden is letting a double be your only DB coverage.

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
| Randomness (`random`, `randomUUID`) | DB adapters/queries — cover with a real test DB |
| File system, sub-processes | Your schemas — use the real schema |
| Browser globals in unit tests | Your handlers — test through the boundary with a real `Request` |
| Logging / error tracker SDKs | Your components — render and assert on DOM |

## Pressure Resistance

### "Mocking is faster than setting up a real DB"

For one test, yes. Across the whole suite, no — real DB tests catch real bugs, and the setup cost is paid once and spread across every test. Modern test runners with prepared fixtures keep DB tests under 100ms each. The "fast" mock test is also the test that lets a column-name typo ship.

### "Mocks isolate the unit-under-test"

Mocks isolate the unit from a port so its decisions can be tested directly. Integration tests then verify that your real adapter satisfies that port. Do not mock the thing the current test claims to verify, and do not confuse a unit double with adapter coverage.

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
- A test hand-monkey-patches `Date.now` (e.g. `Date.now = () => ...`) instead of using the runner's clock control or a `now` seam.
- A mocked query builder is the only test of database behavior.
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
