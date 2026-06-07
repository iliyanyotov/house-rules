---
name: seams-for-untestable-code
description: Use when a function under test imports the DB, `fetch`, `Date`, or a third-party SDK directly and you can't write a unit test for it without mocking three modules. Use when "this is hard to test" comes up on legacy code. Use when tempted to refactor a 300-line function before testing it. Use when the right answer feels like "rewrite the whole module" — there's almost always a smaller intervention.
---

# Seams for Untestable Code

## Overview

**When legacy code can't be tested because of a hard dependency, introduce a *seam* — the smallest possible point where you can substitute a test double — at one well-chosen place.** Don't refactor the entire module first. The seam is the change; the test is the goal; large refactors come later, with the test as safety net.

A **seam** is a place where program behavior can be changed *without editing in that place*. A function parameter is a seam. A constructor argument is a seam. The point is **minimal intervention**: shift just enough so a test double can take the dependency's place.

This skill is specifically for **legacy rescue** — code that already exists and resists being tested. For greenfield code where you're designing modules from scratch, the related discipline is *dependency-inversion* (the domain declares ports; adapters implement them). Seams retrofit testability onto code that wasn't built for it.

## The Iron Rule

```
NEVER refactor untestable legacy code before testing it. Introduce the smallest seam, test, then refactor.
```

**No exceptions:**
- Not for "adding a parameter changes the signature"
- Not for "this is a lot of indirection just for tests"
- Not for "I'll just mock the import"
- Not for "I don't want to refactor before fixing the bug"

## Why

When you confront a function that imports the database, calls `fetch(...)`, and reads `Date.now()` mid-body, the temptation is to declare it "untestable" and either skip the test or refactor the whole thing first. Both options fail:

- **Skip the test:** the bug ships. Or, more likely, the fix you're about to make breaks something else and you find out from production.
- **Refactor first:** you change a lot of code without a test, exactly the failure mode the test was meant to prevent. The refactor introduces bugs the (still missing) tests would have caught.

The seam pattern is the third path: **change the minimum needed to write the test, then test, then make your actual change**. The function might still look mostly untestable to someone reading it cold — that's fine. The narrow seam you introduced is enough.

This skill makes TDD possible on real codebases and pairs tightly with `characterization-tests-first-for-legacy` (you can't characterize what you can't isolate).

## How this differs from dependency-inversion

Both skills are about not coupling code to hard dependencies, but they apply at different times:

- **Dependency-inversion** is the *greenfield* discipline. The domain declares the ports it needs (`type ProductReader = { findById: ... }`); adapters in the outer layer implement them. Designed-in from day one.
- **Seams for untestable code** is the *legacy-rescue* discipline. Code already exists with hard dependencies; you can't redesign it. You introduce the *smallest possible* seam (often just one parameter on one function) to make it testable, then test, then optionally refactor toward the cleaner design later.

If you're writing new code, reach for dependency-inversion. If you're rescuing existing code, seams are the cheaper move.

## Detection

You are violating the rule if any of these are true:

- A function imports a side-effecting module at the top and calls it inside its body, with no way to substitute the import for a test.
- A "test" relies on a real network, real DB, real filesystem, real wall-clock time.
- A test uses module-mocking (`vi.mock()`, `jest.mock()`) to replace 3+ modules per file.
- A test sets `globalThis.fetch = mockFn` or `Date.now = () => fixedTime` as setup.
- A unit test takes more than ~50ms to run.
- A bug fix is blocked on "we'd need to refactor a lot first."
- The phrase "it's tested in integration" appears for code where unit-level coverage would be cheaper.

## The Pattern

### Three kinds of seam, ordered from cleanest to ugliest

1. **Function-parameter seam** (cleanest) — add a parameter for the dependency; default it to the production value in the wrapper that production uses.
2. **Constructor / closure seam** — for class methods or factory functions, accept the dependency at construction.
3. **Module-import seam** (ugliest, last resort) — use module-mocking to swap the imported value at test time. Only when 1 and 2 aren't feasible (framework-bound code: route handlers, framework hooks).

### Function-parameter seam

```ts
// ❌ Untestable: depends on real Date and real db.
export async function findStaleSessions() {
  const cutoff = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  return db.select().from(sessions).where(lt(sessions.lastSeenAt, cutoff));
}
```

```ts
// ✅ Inject the clock and the DB at the function boundary.
//    Production wires the real ones; tests pass fakes.
type DbLike = Pick<typeof db, 'select'>;

export async function findStaleSessions(
  now: Date,
  database: DbLike = db,
) {
  const cutoff = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000);
  return database.select().from(sessions).where(lt(sessions.lastSeenAt, cutoff));
}
```

Production:

```ts
const stale = await findStaleSessions(new Date());
```

Test:

```ts
test('finds sessions older than 7 days', async () => {
  const fakeDb = {
    select: () => ({ from: () => ({ where: () => Promise.resolve([{ id: 's1' }]) }) }),
  };
  const result = await findStaleSessions(new Date('2026-05-17'), fakeDb);
  expect(result).toEqual([{ id: 's1' }]);
});
```

Two seams (the `now` parameter and the `database` parameter). The function body is untouched aside from reading from the parameters. The change is minimal.

`Pick<typeof db, 'select'>` is the trick that makes this cheap: TS lets you describe the *shape* of the dependency as "just the methods I call," so the test fake doesn't have to mock the entire database surface.

### Constructor / closure seam

For class-based code (or factory-style code with persistent state), accept the dependency once at construction:

```ts
// ❌ Hard-coded dependency.
export class EmailService {
  async send(input: ContactFormData) {
    const client = new MailClient(env.MAIL_API_KEY);
    return client.emails.send({ /* ... */ });
  }
}
```

```ts
// ✅ Inject the SDK; tests provide a fake.
type MailLike = { emails: { send: (msg: SendArgs) => Promise<SendResult> } };

export class EmailService {
  constructor(private readonly client: MailLike) {}
  async send(input: ContactFormData) {
    return this.client.emails.send({ /* ... */ });
  }
}
```

Production:

```ts
export const emailService = new EmailService(new MailClient(env.MAIL_API_KEY));
```

Test:

```ts
test('sends with the expected from address', async () => {
  const calls: SendArgs[] = [];
  const service = new EmailService({
    emails: { send: async (msg) => { calls.push(msg); return { id: '1' }; } },
  });
  await service.send({ fullName: 'A', email: 'a@b.c', message: 'hi' });
  expect(calls[0].from).toBe('noreply@example.com');
});
```

### Module-import seam (last resort)

For framework-bound code (route handlers that you can't add parameters to without breaking the framework contract), the seam is at the module-import level:

```ts
// app/api/contact/route.ts
import { emailService } from '../../../services/email/EmailService';

export async function POST(req: Request) {
  const data = await req.json();
  return emailService.send(data);
}
```

Test (with the test runner's module-mocking API):

```ts
vi.mock('../../../services/email/EmailService', () => ({
  emailService: { send: vi.fn(async () => ({ ok: true })) },
}));

const { POST } = await import('./route');
const res = await POST(new Request('https://example/api/contact', {
  method: 'POST',
  body: '{}',
}));
```

Module-import seams are *ugly* — they're action-at-a-distance, they leak across test files if you don't restore them, and they make the test setup hard to read. **Use them only when 1 and 2 aren't options.** When the choice is between writing a test via module-mocking or writing no test, the module-mock wins.

### Strangler fig: introducing seams to a large function

Sometimes the function is 200 lines and you only need to change line 137. You don't want to refactor the function — but the line you need is buried.

**The pattern:** extract the smallest possible block around line 137 into its own function, pass the dependencies it needs as parameters, and call the new function from the original. The new function has a seam; the original is unchanged in shape.

```ts
// ❌ A 200-line function where you need to test the discount logic.
async function processOrder(order: Order) {
  // ... 100 lines ...
  const discount = order.code.startsWith('SAVE') ? order.total * 0.9 : order.total;
  // ... 100 more lines ...
}
```

```ts
// ✅ Extract the discount logic — testable in isolation now.
export function applyCodeDiscount(total: number, code: string): number {
  return code.startsWith('SAVE') ? total * 0.9 : total;
}

async function processOrder(order: Order) {
  // ... 100 lines ...
  const discount = applyCodeDiscount(order.total, order.code);
  // ... 100 more lines ...
}
```

The original function still has the same hard dependencies; you didn't tame all of them. You just carved out the one piece you needed to test. The other 199 lines remain "untestable at the unit level" — that's fine, they'll wait their turn.

## Pressure Resistance

### "Adding a parameter changes the function signature"

For internal functions, yes — and it's almost always fine. For exported APIs, default the new parameter to the production value so existing callers don't break: `(now: Date = new Date(), db = realDb)`. The seam is invisible to current callers.

### "This is a lot of indirection just for tests"

The indirection costs: one parameter, one default value. The benefit: every future test, every characterization, every refactor under safety net. The ratio is heavily one-sided.

### "Dependency injection frameworks are overkill"

We're not talking about a DI framework. We're talking about *function parameters*. TS makes this cheap because the parameter type is a `Pick<>` of just what's used — no interface declarations, no container, no annotations. The "DI framework overkill" objection conflates DI containers with the pattern of passing dependencies in.

### "I'll just mock the import"

When you can. When you can't avoid it (framework boundaries), that's the right call. When you *can* parameterize, that's almost always better — module mocks leak across test files, fail silently when typos exist, and require restoring in afterEach. Parameter seams have none of those problems.

### "I don't want to refactor before fixing the bug"

The seam *isn't* a refactor in the dangerous sense. Extracting one line into a parameter is a mechanical transformation that the type system + tests confirm. It's the minimum change needed to test; that's why it's safe. The big refactor — the one you're afraid of — comes *after* the test, with the test as the safety net.

### "The function takes 10 parameters now"

If your seam-extraction produced a 10-parameter function, that's a *signal* the function had 10 hidden dependencies and was doing too much. The 10 parameters made the design problem visible. The next step is to split. But the seam was the right first move — the alternative was leaving the 10 dependencies hidden.

## Red Flags

- A test file with module-mocks for 3+ modules.
- A unit test that takes >50ms because it's hitting a real dependency.
- A function that imports a database client, `fetch`, or a third-party SDK and uses it directly, with no parameter alternative.
- A "test" that's actually an integration test wearing a unit-test label (slow, real network, can't run offline).
- A bug-fix PR description that includes "couldn't add a test for this."
- A 200-line function with no callers in tests.
- The phrase "this requires a big refactor first."

**All of these mean: a seam is missing — find the smallest possible one and introduce it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Parameters bloat the signature" | One parameter to test = one parameter saved every time the test runs (vs. mock-module setup). |
| "We can't change the API" | Default the new parameter; existing callers are unaffected. |
| "Mocking the import is easier" | Easier to *write*; harder to *trust*. Module mocks leak. |
| "The function is too tangled to seam cleanly" | Then introduce one seam *anyway* at the smallest place. Don't fix everything at once. |
| "We'd need a DI framework" | We need a function parameter. That's all. |
| "Integration tests are enough" | They're enough for confidence; they're not enough for fast feedback during dev. |

## Reference

- Michael Feathers, *Working Effectively with Legacy Code* (2004) — the canonical book on this discipline. The chapter on Seams is the source. Feathers names four seam types; in modern JS/TS the practical taxonomy collapses to the three above (parameter, constructor, module-import).
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (2009) — frames the same idea from the *forward* direction: when writing new code, design it with seams from the start. Their term is "ports" — interfaces that the domain depends on, with adapters that implement them.
