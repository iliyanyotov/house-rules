---
name: decouple-deploy-from-release-via-flags
description: Use when shipping a risky feature — schema migration, payment integration, third-party SDK upgrade, UI rewrite. Use when "we'll just revert if it breaks" is the rollback plan for a feature that can't be cleanly reverted (data migrations, public URL changes). Use when a PR's blast radius is "every user immediately." Use when reviewing code that gates new behavior behind a one-off env var (`ENABLE_NEW_FLOW`) without a typed flag system.
---

# Decouple Deploy from Release via Flags

## Overview

**Code is deployed; *features* are released.** A deploy is the platform-mediated act of getting binaries into production; a release is the operator-mediated act of letting users see new behavior. The two must be separable, so that risky behavior can be turned on by a flag flip (seconds, reversible) instead of a re-deploy (minutes, requires CI, and may carry unrelated changes).

The release surface is a **typed flag** (string-literal union or branded variant), not a free-form env var. The flag has documented states, a default, an owner, and a removal plan.

## The Iron Rule

```
NEVER couple a risky behavior change to its deploy. Ship the code dark; release with a flag flip.
```

**No exceptions:**
- Not for "we can just revert if it breaks"
- Not for "rollback is fast enough"
- Not for "small team, no need"
- Not for "we don't need shadow / canary; just turn it on"

## Why

A deploy mixes two concerns that have different risk profiles:

- **Infrastructural**: this binary will run, this schema will be in place, these env vars are set. *Atomic.*
- **Behavioral**: users will start seeing this new flow, this new payment integration will charge real cards, this rewritten checkout will fire. *Not* atomic — it affects every user the moment the deploy promotes.

Coupling them means every deploy carries every new behavior. The blast radius of a deploy is the *union* of everything in it. A small fix to a logging line shipped in the same deploy as a risky payment integration carries the payment risk. To roll back the payment, you also roll back the logging fix. To roll back without the payment going live, you have to re-deploy without it — but re-deploy includes the *current* main, which now has the next person's commits.

Flags decouple these. The deploy is benign — new code is in production but the new behavior is *off*. The release is the flag flip — a few users at a time, then all users, with the ability to flip back to *off* in seconds without a re-deploy.

This is core to the DORA framing of "deployment frequency" being a *leading* indicator of organizational health: teams that decouple deploy from release deploy more often *and* break things less often, because the deploys are smaller, mostly-benign, and recoverable independent of CI.

## Detection

You are violating the rule if any of these are true:

- A risky feature ships by re-deploying main with the feature on; rollback plan is "revert and re-deploy."
- A feature flag is a free-form env var like `ENABLE_NEW_CHECKOUT=true` with no typed wrapper and no enumerated states.
- A flag is set in code (`const USE_NEW_FLOW = true;`) and "released" by editing the constant and re-deploying.
- A flag has no documented owner, no removal plan, and has been in the codebase >6 months.
- "Rollback" of a new feature requires editing data (because the old code can't read the new shape).
- A feature was reverted by re-deploying an older commit, and that re-deploy lost an unrelated fix.
- The codebase has 20+ accumulated flags from old launches, none of them removed.

## The Pattern

### A typed flag, not a free-form env var

```ts
// ❌ Free-form, no enumeration, easy to typo.
const useNewCheckout = process.env.ENABLE_NEW_CHECKOUT === 'true';

// ✅ Typed, enumerated, defaulted.
type CheckoutMode = 'classic' | 'v2-shadow' | 'v2-canary' | 'v2-full';

const checkoutMode: CheckoutMode = (() => {
  const raw = process.env.CHECKOUT_MODE;
  switch (raw) {
    case 'classic':
    case 'v2-shadow':
    case 'v2-canary':
    case 'v2-full':
      return raw;
    default:
      return 'classic'; // safe default
  }
})();
```

The typed union forces the consumer to handle every state via exhaustive switch. Adding a state is a deliberate code change; setting `CHECKOUT_MODE=v3` accidentally falls back to `classic` rather than corrupting behavior.

### Five-stage rollout — the canonical sequence

A risky feature passes through these stages:

```
classic        — old code path is the only path. New code is deployed but unused.
v2-shadow      — both paths run; new path's output is logged & compared, not user-visible.
v2-canary      — small % of traffic uses the new path (e.g., internal users, 1% of customers).
v2-canary-50   — 50% of traffic.
v2-full        — 100% of traffic. Old code path remains for ~one deploy as a safety net.
[cleanup]      — old code deleted; flag removed; type narrows to just 'v2-full'.
```

Each stage is a flag flip — seconds, no deploy. If a stage misbehaves, flip back. The deploy that introduced the new code path stays out — only the flag changes.

### Consuming the flag — discriminated branch with shared parsing

```ts
// ❌ Boolean if/else — easy to drift; hard to A/B.
if (useNewCheckout) {
  return placeOrderV2(input);
} else {
  return placeOrderV1(input);
}

// ✅ Exhaustive over the flag's union.
async function placeOrder(input: CheckoutInput): Promise<Order> {
  switch (checkoutMode) {
    case 'classic':
      return placeOrderV1(input);
    case 'v2-shadow': {
      const real = await placeOrderV1(input);
      // fire-and-forget compare; never fails the user
      placeOrderV2Shadow(input)
        .then((shadow) => logShadowDiff(real, shadow))
        .catch((err) => log.warn('shadow_v2_failed', err));
      return real;
    }
    case 'v2-canary':
    case 'v2-canary-50':
    case 'v2-full':
      return placeOrderV2(input);
    default: {
      const _exhaustive: never = checkoutMode;
      throw new Error(`unhandled checkout mode: ${String(_exhaustive)}`);
    }
  }
}
```

The exhaustive switch guarantees every flag state is handled. Removing a state at cleanup time becomes a compile-error-driven refactor — every consumer is forced to acknowledge the change.

### Where the flag value comes from

Three options in order of capability:

1. **Env var** (`CHECKOUT_MODE`) — simplest. Flipping requires a platform env change + re-deploy of *one* env value, which is faster than a full code re-deploy but not instant.
2. **Low-latency KV store / config service** — flags live in a sub-millisecond fetch. Flipping is *instant* — no deploy. Good for runtime-toggleable flags.
3. **Dedicated flag platform** — adds per-user targeting, gradual rollout, contextual rules. Worth it once you have multiple flags in flight.

The skill applies regardless of the source — what matters is the typed boundary into the application.

### Document the flag

Every flag has a short doc — in code or in a flag-platform UI:

```ts
/**
 * CHECKOUT_MODE — controls the checkout pipeline.
 *
 * States:
 *   - classic    → original code path
 *   - v2-shadow  → both paths run; v2 output compared/logged
 *   - v2-canary  → 1% of traffic uses v2 (set in config rule)
 *   - v2-full    → 100% of traffic
 *
 * Owner: payments team (@alice)
 * Created: 2026-04-12
 * Target removal: 2026-06-01 (after 1 month at v2-full)
 * Tracking issue: #1842
 */
```

The metadata isn't ceremony — it's the *only* thing that prevents the flag from becoming permanent debt.

### Removal is part of the plan

Every flag has a documented end state. After the feature is fully released and stable for ~one cycle (two weeks, a month — set explicitly), the flag and the old code path are *deleted*.

The flag-removal PR:

```ts
// Before
async function placeOrder(input: CheckoutInput): Promise<Order> {
  switch (checkoutMode) {
    case 'classic':       return placeOrderV1(input);   // delete
    case 'v2-shadow':     /* ... */                       // delete
    case 'v2-canary':
    case 'v2-canary-50':
    case 'v2-full':
      return placeOrderV2(input);
    /* ... */
  }
}

// After
async function placeOrder(input: CheckoutInput): Promise<Order> {
  return placeOrderV2(input);
}
```

Plus: delete `placeOrderV1`, delete the `CheckoutMode` type, delete the env var. Ship as its own small PR.

### Composition with expand/contract migrations

```
Deploy 1 — expand schema; new code path behind flag, default off
Deploy 2 — flag rollout: off → shadow → canary → full
Deploy 3 — contract schema; remove old code path; remove flag
```

The flag turns three deploys' worth of risk into three *small* deploys, each individually reversible. The schema migration is gated by the same flag that controls the code path — no race between "schema is ready" and "code uses it."

## Pressure Resistance

### "Flags are over-engineering for a small team"

The smallest flag system is a typed env var. That's one file (the parsing wrapper) and zero infrastructure. It still gives you the deploy/release split. The over-engineered version (per-user targeting + variants) is for *later*. The *discipline* of decoupling is for *now*.

### "We can just revert if it breaks"

Reverting works if (a) the bad commit is the only thing on main and (b) the change has no data side effects. Both conditions rarely hold. A flag flip is faster, narrower, and side-effect-free.

### "Platform rollback is fast enough"

A platform rollback rolls back *the whole deploy* — including unrelated fixes shipped alongside. That's the coupling this rule is breaking. The flag flip rolls back *only the misbehaving feature*.

### "Flags add complexity to the code"

A `switch` on a typed union is the same shape as any discriminated state we already use. The complexity is bounded, named, and removed when the flag is cleaned up. The alternative complexity (re-deploys, multi-step coordinated launches) is *worse* and *unbounded*.

### "We don't have flag infrastructure"

You have env vars. Start there. Migrate to a config service when you need runtime flip; migrate to a flag platform when you need per-user targeting. The discipline scales independently of the tooling.

### "Flags become permanent debt"

Only if you let them. The rule's removal-plan half is *equally important* as the introduction half. Document the target removal date in the flag's comment; track it in your issue tracker; treat a 6-months-old flag as a bug.

### "We need the new feature on for everyone immediately"

Then the flag's stages compress — shadow for one day, canary for one day, full. The discipline is still the same; the speed varies. *Some* gap between deploy and release is what gives you the ability to flip back without a re-deploy.

## Red Flags

- A "feature flag" that's actually `process.env.ENABLE_X === 'true'` with no typed wrapper.
- A PR description that includes "deploy plan: announce in chat, then merge and watch metrics."
- A boolean flag that has two states forever (no shadow / canary / cleanup planning).
- A flag that's been in the codebase >6 months with no removal plan.
- A flag whose default is on but the code still has the off branch — old branch is dead code.
- A migration PR with no flag — the change goes from "off" to "on for 100% of users" the moment the deploy lands.
- 10+ active flags with no inventory or owners.
- The phrase "we'll just revert" as the rollback plan for a feature with data side effects.
- A feature where rollback requires editing data because the old code can't read the new schema.

**All of these mean: deploy and release are coupled — the next failure won't have a flip-back path.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Small team, no need" | Smaller teams break things faster *and* recover slower. Flags help small teams more than large ones. |
| "Platform rollback is fast" | Rollback is whole-deploy. Flags are per-feature. Different granularities. |
| "Flags add complexity" | Bounded, named, removable complexity. The alternative is unbounded coordination cost. |
| "We don't need shadow / canary; just turn it on" | Sometimes true. Then the flag's stages compress to two (off / full). The *option* to canary is what matters. |
| "Flags will pile up" | Only if you let them. The rule's removal half prevents this. |
| "Reverting is good enough" | Reverting reverts *everything*. Flags revert *one thing*. Different scopes. |

## Reference

- Forsgren, Humble, Kim, *Accelerate* (2018) — the DORA finding that *deployment frequency* and *change failure rate* both improve when deploys and releases are decoupled.
- Jez Humble & David Farley, *Continuous Delivery* (2010), ch. 10 — "deployment pipeline" architecture explicitly separates *deploy* from *release*. Feature flags are the operational mechanism.
- Martin Fowler, ["Feature Toggles"](https://martinfowler.com/articles/feature-toggles.html) — the canonical write-up. Distinguishes release toggles, experiment toggles, ops toggles, and permission toggles.
- Pete Hodgson, ["Feature toggles are one of the worst kinds of technical debt"](https://www.startuprocket.com/articles/feature-toggles-are-one-of-the-worst-kinds-of-technical-debt) — the cautionary version. Flags are a *tool*; without removal discipline, they're debt.
