---
name: test-observable-behavior-not-implementation
description: Use when writing a unit test and choosing what to assert. Use when a refactor breaks tests even though external behavior didn't change. Use when reviewing a PR whose tests are mostly `expect(spy).toHaveBeenCalledWith(...)`. Use when the first instinct is to spy on a private helper. Use when the test setup is longer than the production code it tests.
---

# Test Observable Behavior, Not Implementation

## Overview

**A test asserts what the *caller* can observe** — return values, persisted rows, sent messages, rendered DOM, thrown errors. Not which internal helper ran, in which order, with which intermediate values.

Refactor-resistance is the test for the test: if you can restructure the unit's internals without breaking the test, the test asserts behavior. If a no-op rename breaks it, the test asserts implementation.

## The Iron Rule

```
NEVER assert on internal helpers, private methods, or call order — only on observable outcomes.
```

**No exceptions:**
- Not for "I have to test the helper somewhere"
- Not for "mocking is the only way to make this fast"
- Not for "100% line coverage requires it"
- Not for "London-school says mock everything"

## Why

A test exists to answer one question: *if I change this code, will I find out that I broke something?* The value of a test is the bugs it catches **minus** the false alarms it raises during refactors.

Implementation-detail tests have negative ROI as the code grows. Every internal restructure (renaming a helper, inlining a function, changing the order of two pure steps) breaks tests that asserted on internals. The bugs were never real. The tests still cost time to fix and trust to maintain. Eventually the team stops believing test failures *mean* anything, and a real regression slips through.

Behavior tests survive refactors. Same fixture in → same observable out. The implementation can be rewritten three times under the same test and the test still tells the truth.

## Detection

You are violating the rule if any of these are true:

- A test calls `spyOn` on a function defined in the same module being tested.
- A test asserts `expect(spy).toHaveBeenCalledWith(...)` on an internal helper.
- A test inspects a private field or non-exported variable.
- A test must be edited when an *internal-only* rename happens.
- A test's setup constructs more mocks than the unit has external dependencies.
- A test asserts on the *order* internal helpers ran.

## The Pattern

The "through the door" lens — imagine the unit has one front door (the exported function), and the rest is opaque. What can you observe from outside?

- What it **returns** (or throws)
- What it **writes** — to the DB, to an HTTP call out, to a queue
- What it **renders** (for components, the DOM)
- What it **passes** to dependencies you control as the test author (the injected seams)

That list is the assertion surface. Anything else is implementation.

### Return value as the observable

```ts
// account.ts
export function deposit(account: Account, amount: Money): Account {
  if (amount.cents <= 0) throw new Error('non_positive_amount');
  const updated = addMoney(account.balance, amount); // internal helper
  return { ...account, balance: updated, lastActivityAt: now() };
}

// ❌ Asserts on the helper. Inline `addMoney` → test breaks for no real reason.
test('deposit calls addMoney with the right args', () => {
  const spy = spyOn(money, 'addMoney');
  deposit(account, money(100));
  expect(spy).toHaveBeenCalledWith(account.balance, money(100));
});

// ✅ Asserts on the observable outcome — the new account's balance.
test('deposit increases the balance by the deposited amount', () => {
  const before = makeAccount({ balanceCents: 500 });
  const after = deposit(before, money(100));
  expect(after.balance.cents).toBe(600);
});

// ✅ Asserts on the contract violation — observable as a throw.
test('deposit rejects non-positive amounts', () => {
  expect(() => deposit(makeAccount(), money(0))).toThrow('non_positive_amount');
});
```

### Side effect at the injected seam — the side effect *is* the observable

```ts
// notifier.ts — orchestrator with one side effect
export async function notifyOrderShipped(
  order: Order,
  deps: { send: SendFn },
) {
  if (!order.customerEmail) return;
  await deps.send({
    to: order.customerEmail,
    template: 'shipped',
    data: { orderId: order.id },
  });
}

// ✅ The injected `send` is the legitimate seam — assert on what was sent.
test('notifyOrderShipped sends a shipped-template message to the customer', async () => {
  const calls: SendArgs[] = [];
  const send: SendFn = async (args) => { calls.push(args); };

  await notifyOrderShipped(makeOrder({ customerEmail: 'a@b' }), { send });

  expect(calls).toHaveLength(1);
  expect(calls[0]).toMatchObject({ to: 'a@b', template: 'shipped' });
});

// ✅ Negative case — also observable (absence of call).
test('notifyOrderShipped skips when customerEmail is missing', async () => {
  const calls: SendArgs[] = [];
  await notifyOrderShipped(makeOrder({ customerEmail: null }), { send: async (a) => { calls.push(a); } });
  expect(calls).toHaveLength(0);
});
```

The mock is on a dependency *injected from outside*, not on an internal helper. That's the legitimate seam.

### Persisted state is observable

```ts
// ✅ Assert on the DB row, not on the query builder.
test('createOrder persists the order with status=pending', async () => {
  const orderId = await createOrder({ customerId, items });
  const persisted = await orders.findById(orderId);
  expect(persisted).toMatchObject({ id: orderId, status: 'pending' });
});

// ❌ Spying on the insert call. Breaks if you switch query libraries.
test('createOrder calls db.insert with orders table', async () => {
  const spy = spyOn(db, 'insert');
  await createOrder({ customerId, items });
  expect(spy).toHaveBeenCalledWith(orders);
});
```

The behavior test stays green when you change query libraries or column order. The implementation test fails on both.

### DOM is observable, state is not

```ts
// ❌ Reading component internals — testing useState.
test('clicking add increments count state', () => {
  const { result } = renderHook(() => useCounter());
  act(() => result.current.add());
  expect(result.current.internalCount).toBe(1);
});

// ✅ Reading what the user sees — testing the rendered DOM.
test('clicking add increments the displayed count', async () => {
  render(<Counter />);
  await userEvent.click(screen.getByRole('button', { name: 'Add' }));
  expect(screen.getByText('1')).toBeInTheDocument();
});
```

`internalCount` is a refactor target. The displayed `1` is what the user came for.

## Pressure Resistance

### "But I need to verify the helper is called"

Why? If the helper's effect is observable, assert on the effect. If it isn't, it's implementation — moving it around shouldn't break tests. If the helper *is* the seam to an external system, inject it from outside and assert on the injected double.

### "Mocking is the only way to make this fast"

Sometimes true. The mock then lives at the *architectural boundary* — your code, their code. You don't mock your own helpers; you mock the network call your helper makes. The unit calls the boundary; the test substitutes the boundary; the test asserts what the unit did *given* the boundary's response.

### "100% line coverage requires mocking internals"

100% line coverage is a vanity metric. Coverage that requires implementation-detail assertions costs more than it saves.

### "How do I test private logic, then?"

You don't — directly. Private logic is exercised *through* the public interface. If you can't reach a private branch through any public input, the branch is dead code or the public interface is too narrow.

### "The behavior assertion is awkward — the helper's args are easier"

The helper's args are *closer* to your fingers — they're inside the same module. They're also coupled to today's implementation choice. "Easy to write" is not the right axis; "still meaningful after a refactor" is.

## Red Flags

- A test file that imports private helpers from the module-under-test.
- `spyOn(thisModule, 'someHelper')` patterns.
- A test that breaks when you change `for` to `forEach` in the implementation.
- A test that asserts `toHaveBeenCalledTimes(1)` on an internal pure function.
- A test that mocks more deps than the unit imports.
- A test suite where every refactor PR has a "fix tests" commit that doesn't fix a real defect.

**All of these mean: the test is asserting implementation, not behavior — rewrite it.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "I have to test the helper *somewhere*" | If it has observable behavior, it has its own public interface — test it through that. If not, it's implementation. |
| "Mocking the DB is too slow without spies" | Use a real test DB (containers, in-memory variants, ephemeral branches). The DB *is* the observable. |
| "The test documents the implementation" | Code documents the implementation. Tests document the contract. |
| "My team mocks everything; it's the convention" | Conventions are revisable. Start writing behavior tests for new code. |
| "How else do I get to 100% coverage?" | You don't. Aim for behavior coverage, not line coverage. |
| "Snapshot tests are behavior tests" | Only if the snapshot captures something the user observes. A snapshot of an internal data structure isn't. |

## Reference

- Vladimir Khorikov, *Unit Testing: Principles, Practices, and Patterns* (2020), ch. 4–5 — the modern canonical articulation. The four pillars: protection against regressions, refactor resistance, fast feedback, maintainability.
- Kent Beck, *Test-Driven Development by Example* (2002) — "test through the door"; the original framing of tests written against external behavior.
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (2009) — "listen to your tests"; mock at the right seams (adapters), not at internals.
- Gary Bernhardt, "Boundaries" (2012) — functional core / imperative shell as the architectural pattern that makes behavior testing trivial.
