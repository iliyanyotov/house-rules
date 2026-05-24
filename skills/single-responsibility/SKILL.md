---
name: single-responsibility
description: Use when designing a module, file, function, or component. Use when a description of what something does requires the word "and." Use when reviewing code that mixes transport (HTTP/DB), domain logic, and presentation in one place. Use when a component renders, fetches, validates, and persists. Use when a route handler does parsing, business logic, DB writes, email sends, and response shaping inline.
---

# Single Responsibility

## Overview

**A module has one reason to change.** If you need the word "and" to describe what it does, split it.

A reason to change is a *stakeholder concern* — a customer-facing requirement, a schema, a UI layout, an auth policy. Two stakeholders, two modules.

## The Iron Rule

```
NEVER let a module own two reasons to change. If "and" describes it, split it.
```

**No exceptions:**
- Not for "splitting means jumping between files"
- Not for "it's only used once"
- Not for "the team will refactor later"
- Not for "this is the same domain so it belongs together"

## Why

Code with one reason to change is small, named, and obvious. Code with three reasons is large, vaguely named, and constantly touched. Every commit risks bugs in the unrelated concerns living next door.

An AI assistant can rewrite a 40-line single-purpose function without breaking anything; it cannot safely rewrite a 400-line function that touches HTTP, validation, persistence, and email — too many things move at once. Small modules are agent-safe.

## Detection

You are violating the rule if any of these are true:

- A file's name doesn't predict what's inside (`utils.ts`, `helpers.ts`, `lib.ts`).
- A function's name uses `And` — `validateAndSave`, `fetchAndRender`, `parseAndStore`.
- A handler does parsing, business logic, persistence, email sends, and response shaping.
- A class has methods that don't share state — a junk drawer with `this`.
- A test for the module must mock four unrelated systems.
- A bug fix in one feature requires editing a file owned by another feature.

## The Pattern

### Handler — one responsibility: the HTTP boundary

```ts
// ❌ Doing everything inline. Hard to test, hard to reuse.
export async function handleRegister(rawBody: unknown) {
  const body = rawBody as { email?: string };
  if (!body.email || !body.email.includes('@')) {
    return { status: 400, body: { error: 'bad_email' } };
  }
  const [user] = await users.insert({ email: body.email }).returning();
  await mailer.send({ to: body.email, subject: 'Welcome', html: '<p>Hi</p>' });
  return { status: 201, body: { user } };
}

// ✅ Each layer has one job.

// 1. Parse — schema only.
const RegisterInput = z.object({ email: z.string().email() });

// 2. Domain operation — persistence.
async function registerUser(input: z.infer<typeof RegisterInput>): Promise<User> {
  const [user] = await users.insert(input).returning();
  return user;
}

// 3. Side effect — its own module.
async function sendWelcomeEmail(user: User): Promise<void> {
  await mailer.send({ to: user.email, subject: 'Welcome', html: renderWelcome(user) });
}

// 4. Handler — just the boundary.
export async function handleRegister(rawBody: unknown) {
  const parsed = RegisterInput.safeParse(rawBody);
  if (!parsed.success) return { status: 400, body: { error: 'bad_input' } };

  const user = await registerUser(parsed.data);
  await sendWelcomeEmail(user);
  return { status: 201, body: { user } };
}
```

Each piece is independently testable, replaceable, and named.

### Component — one rendering concern

```tsx
// ❌ Component that does too much.
function OrderList() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);
  const [filter, setFilter] = useState('');
  useEffect(() => {
    setLoading(true);
    fetch('/api/orders').then(r => r.json()).then(data => {
      setOrders(data); setLoading(false);
    });
  }, []);
  const filtered = orders.filter(o => o.customer.includes(filter));
  if (loading) return <Skeleton />;
  return <>
    <input value={filter} onChange={e => setFilter(e.target.value)} />
    <ul>{filtered.map(o => <li key={o.id}>{o.customer} — ${o.totalCents / 100}</li>)}</ul>
  </>;
}

// ✅ Separated by concern.
function useOrders() { /* fetch hook */ }
function filterByCustomer(orders: readonly Order[], q: string) { /* pure */ }
function OrderRow({ order }: { order: Order }) { /* presentation only */ }

function OrderList() {
  const [filter, setFilter] = useState('');
  const { data: orders = [], isLoading } = useOrders();
  if (isLoading) return <Skeleton />;
  const filtered = filterByCustomer(orders, filter);
  return <>
    <input value={filter} onChange={e => setFilter(e.target.value)} />
    <ul>{filtered.map(o => <OrderRow key={o.id} order={o} />)}</ul>
  </>;
}
```

The pure function is tested in three lines. The hook is mocked once. The row is a snapshot. The composition is what changes when the page evolves.

### What this rule is *not*: orchestration vs. inlining

Single-responsibility is **one reason to change**, not **few lines**. A 200-line handler that reads top-to-bottom as `auth → idempotency → parse → call service → log → return` is *operationally coherent*. Its one reason to change is "the workflow at this URL." Each step is a named call into a service with its own responsibility.

```ts
// ✅ Long handler, but each step delegates. One concern: the workflow.
export async function handleCreateOrder(req: Request) {
  if (!authorize(req)) return unauthorized();

  const idempotencyKey = req.headers.get('idempotency-key');
  const prior = await idempotencyKeys.lookup(idempotencyKey);
  if (prior) return replay(prior);

  const parsed = OrderInput.parse(await req.json());
  const order = await orderService.create(parsed);
  await notificationService.sendConfirmation(order);
  await analytics.track('order_created', order.id);

  return Response.json({ order });
}

// ❌ Same line count, but logic is inlined. Three concerns: validation, persistence, email — all in the handler.
export async function handleCreateOrder(req: Request) {
  const body = await req.json();
  if (!body.customerId || typeof body.totalCents !== 'number') return badRequest();
  const [order] = await orders.insert(body).returning();
  await mailer.send({ to: body.customerEmail, subject: 'Confirmed', html: `Order ${order.id}` });
  return Response.json({ order });
}
```

Both are similar line counts. Only one is a violation. The question is *what does this module itself do*, not *how many lines*.

## Pressure Resistance

### "Splitting means jumping between files"

Each file is then small enough to hold in your head. The alternative is one giant file you can't hold anywhere. Cognitive load of "what does this 400-line function do" is much higher than "what does this 40-line function do, and where's the one it calls."

### "It's only used once, why extract it?"

Extraction isn't about reuse; it's about *naming*. A 12-line block named `filterByCustomer` is a label for "this is the filter logic." When inline, you re-parse it every time you read the surrounding function.

### "I'll keep it together and refactor later"

You won't. The combined function develops dependencies on its own internals. By the time anyone tries to extract, the parts have grown together — a much larger refactor than just writing it separately to start.

### "Single-responsibility is OO theory"

The framing is OO; the rule is universal. A pure function has a responsibility. A component has a responsibility. A query helper has a responsibility. The boundary is the *concern*, not the abstraction style.

### "But I had to add `And` to handle a transaction"

Transactions are a separate concern: an *orchestrator* that calls two operations, and each operation does one thing. The orchestrator name reflects the workflow (`registerUserWithWelcome`); the two operations stay named `registerUser` and `sendWelcomeEmail`.

## Red Flags

- Function or file names with `And`, `Plus`, `Util`, `Helper`, `Manager`, `Service` without further specificity.
- A handler longer than ~30 lines of inlined logic (orchestrators that delegate to named services are fine even when long).
- A component longer than ~100 lines.
- A function that imports from four different concerns and *inlines* logic from each.
- A test setup that needs to stub five unrelated modules.
- A bug in feature A that requires editing the file owned by feature B.

**All of these mean: split before the next commit.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Splitting is overkill for small features" | The split happens when the feature grows. Better to split early than extract later. |
| "It's the same domain, so it belongs together" | Same domain includes presentation, persistence, validation — separate concerns. |
| "Refactoring later is easy" | Only if you split it later. If you don't, dependencies compound. |
| "I want to see everything in one place" | One screen-page, not one 400-line function. Vertical scroll isn't a substitute for naming. |
| "Splitting hurts performance" | Modern runtimes inline small functions. Overhead is a non-issue at app-code scale. |
| "Mixing concerns is more pragmatic" | Pragmatism is what the next person will say when debugging your god function at 2am. |

## Reference

- Robert C. Martin, *Agile Software Development, Principles, Patterns, and Practices* (2003) — the "S" in SOLID: *"A module should have one and only one reason to change."*
- David Parnas, ["On the Criteria to Be Used in Decomposing Systems into Modules"](https://www.win.tue.nl/~wstomv/edu/2ip30/references/criteria_for_modularization.pdf) (1972) — the original "information hiding" / one-reason-to-change argument, decades older than SOLID.
