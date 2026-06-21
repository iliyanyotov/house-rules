---
name: composition-root-discipline
description: Use when wiring a dependency-injection graph by hand — a `Context`, `container`, `createApp`, or `buildServices` factory that instantiates every service, repository, and client. Use when that factory has grown past a few dozen entries and the constructor takes 20+ parameters. Use when a "used before it was defined" or circular-dependency error appears in startup wiring. Use when tempted to instantiate a service somewhere other than the one wiring file, or to reach for a reflection-based DI framework "because the wiring is getting big."
---

# Composition Root Discipline

## Overview

**Construct the entire object graph in exactly one place — the composition root — and nowhere else.** Every service, repository, and client is instantiated and connected in a single factory, in dependency order. The rest of the app receives already-built objects; it never instantiates its own collaborators.

`dependency-inversion` tells you *what* to depend on (abstractions, injected). This skill governs *where* the injection is assembled. A hand-wired root is a legitimate, framework-free choice — but only if it stays disciplined.

## The Iron Rule

```
NEVER instantiate a service, repository, or client outside the one composition root.
```

**No exceptions:**
- Not "just this one, it's a leaf with no deps"
- Not "the controller needs a fresh instance"
- Not "it's easier to `new` it inline for now"
- Not "a DI framework would be cleaner" (until the root genuinely no longer scales)

## The Pattern: one root, dependency-ordered, layer-grouped

The composition root is a single factory (often `createContext()` or `Context.fromEnv(env)`): it reads validated config, instantiates every dependency in topological order, and returns one container object. Construction happens once, at boot.

```ts
export class Context {
  constructor(
    // grouped by layer — see sub-rules below
    public env: Env,
    public db: Database,
    public paymentClient: PaymentClient,
    public userRepository: UserRepository,
    public orderRepository: OrderRepository,
    public userReadService: UserReadService,
    public orderCreateService: OrderCreateService,
  ) {}

  static fromEnv(env: Env): Context {
    // infrastructure first
    const db = new Database(env);
    // clients next
    const paymentClient = new PaymentClient(env.paymentApiKey);
    // repositories depend on infra
    const userRepository = new UserRepository(db);
    const orderRepository = new OrderRepository(db);
    // services depend on repositories + clients — ORDER MATTERS
    const userReadService = new UserReadService(userRepository);
    const orderCreateService = new OrderCreateService(orderRepository, paymentClient, userReadService);

    return new Context(
      env,
      db,
      paymentClient,
      userRepository,
      orderRepository,
      userReadService,
      orderCreateService,
    );
  }
}
```

Consumers receive the built graph; they never construct:

```ts
// ✅ Handler reads a ready-made service off the container.
export const orders = (ctx: Context) =>
  router.post('/orders', (req) => ctx.orderCreateService.create(parse(req)));

// ❌ Handler instantiates its own dependencies — now wiring lives in two places.
export const orders = (db: Database) =>
  router.post('/orders', (req) =>
    new OrderCreateService(new OrderRepository(db), new PaymentClient(process.env.KEY!)).create(parse(req)));
```

### Sub-rule 1 — One root, and only one

There is exactly one place where `new SomethingService(...)` appears. Grep for `new .*Service(`, `new .*Repository(`, `new .*Client(` — every hit outside the root (and outside tests) is a violation. Tests are exempt: a test constructing a service with stubs is *not* a second composition root; it's the unit under test wired for isolation.

### Sub-rule 2 — Group by layer, in dependency order

Order top-to-bottom: **config → infrastructure (db, cache, queue) → clients (external adapters) → repositories → services**. Within services, a dependency must be declared before its dependents. This ordering is *load-bearing*, not cosmetic — `const`s are used in declaration order, so a service used before it's defined is a runtime crash. Separate the layers with blank-line bands so a 200-line root stays scannable. Where one service must precede another non-obviously, leave a one-line comment saying so.

### Sub-rule 3 — The root only constructs; it holds no logic

The root's single job is `new` + connect. No branching on request data, no computation, no business rules. The only conditionals allowed are *construction-time* choices (pick a real vs. fake client by env). If you're computing a domain value in the root, a concern has leaked out of a service.

### Sub-rule 4 — Adding a dependency touches exactly the root

A new service is wired in one file, in (typically) two-to-three spots: the factory `new`, the constructor parameter, and the constructor call. Make that a checklist so it's never half-wired. If adding a dependency makes you edit five files, the graph isn't centralized.

### Sub-rule 5 — Graduate to a framework only when the root stops scaling

Hand-wiring beats a reflection/decorator container until the root is genuinely unwieldy — order-of-magnitude hundreds of services, or cycles you can't resolve by hand. *Then* adopt a typed container. Not before: a framework trades explicit, debuggable, compile-time-checked wiring for runtime magic, and you only want that trade once the explicit version actually hurts. (This is `yagni` applied to your own infrastructure.)

## Worked Examples

```ts
// ❌ A second construction site. The "graph" is now smeared across the codebase;
//    nobody can answer "what depends on what?" from one file.
function getMailer() {
  return new Mailer(new TemplateRenderer(), process.env.SMTP_URL!);
}

// ❌ Logic in the root. The root decides a business rule at wire time.
static fromEnv(env: Env): Context {
  const tier = env.region === 'eu' ? premiumPricing : standardPricing; // domain decision — wrong place
  const orderService = new OrderService(tier);
  // ...
}

// ✅ Construction-time choice only — a fake adapter in test, real in prod. This is allowed.
static fromEnv(env: Env): Context {
  const paymentClient = env.useFakePayments
    ? new FakePaymentClient()
    : new PaymentClient(env.paymentApiKey);
  // ...
}

// ✅ Non-obvious ordering documented, not left as a trap.
static fromEnv(env: Env): Context {
  // userReadService must precede orderCreateService — the latter injects the former.
  const userReadService = new UserReadService(userRepository);
  const orderCreateService = new OrderCreateService(orderRepository, userReadService);
  // ...
}
```

## Pressure Resistance

**"It's just one leaf service, constructing it inline is harmless."** It is never one. The first inline `new` legitimizes the second; six months later the object graph has no single source of truth and a dependency swap means hunting construction sites. The rule is binary because the exception is unbounded.

**"The constructor has 40 parameters — that's a code smell, I should split the root."** A wide constructor in a composition root is *expected* and fine; it's a list, not a god class (it has no behavior). What would be a real smell is one *service* taking 20 deps — that's a facade hiding multiple responsibilities, and the fix is splitting the service, not the root. Don't confuse a long wiring list with a complex module.

**"This is exactly what a DI framework solves — let's add one."** A framework hides the ordering and the cycles instead of forcing you to resolve them; it turns a compile-time "you forgot a dependency" error into a runtime one. Hand-wiring is the better default precisely because it's explicit. Adopt a container when the root crosses into the hundreds of entries — a real threshold, not a vibe.

**"Circular dependency — I'll just lazy-resolve it with a getter."** A cycle in the graph is a design signal, not a wiring inconvenience. Two services that need each other usually want a third extracted, or an event/callback seam. Papering over it with lazy lookup hides a coupling problem that will resurface.

**"Tests construct services directly — so the 'one root' rule is already broken."** Tests are not the application's composition root. A unit test wiring a service with stubs is isolating the unit; it has no bearing on the production graph. The rule is about *application* construction.

## Common Mistakes

| Smell | Fix |
|---|---|
| `new XService(...)` outside the root (and not in a test) | Move construction into the root; pass the built instance in. |
| Root branches on request/runtime data | Move the decision into a service; the root only makes wire-time choices. |
| Alphabetized factory body that crashes at boot | Reorder by dependency; declare before use; band by layer. |
| A *service* with 20+ constructor params | Split the service (multiple responsibilities); this is not a root problem. |
| Lazy getter added to dodge a cycle | Break the cycle: extract a third collaborator or invert with a callback/event. |
| Adding one service requires editing five files | Centralize wiring; the graph isn't actually rooted. |
| Reaching for a DI framework at 30 services | Stay hand-wired; revisit only in the hundreds. |

## The Bottom Line

One file builds the graph; everything else receives it. The wiring list gets long — that's the cost of being able to see your whole system's shape in one place, and it's a bargain. Keep it ordered, keep it logic-free, and don't trade it for runtime magic until it genuinely stops scaling.

## Reference

- Mark Seemann, *Dependency Injection Principles, Practices, and Patterns* (2019) — defines the Composition Root pattern: *"A Composition Root is a single, logical location in an application where modules are composed together,"* placed as close as possible to the application's entry point.
- Mark Seemann, ["Pure DI"](https://blog.ploeh.dk/2014/06/10/pure-di/) — the case for hand-wired (container-free) composition as the sound default before reaching for a framework.
