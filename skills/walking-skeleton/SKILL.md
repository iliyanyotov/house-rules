---
name: walking-skeleton
description: Use when starting a new feature or a new project. Use when the team has been "building the foundation" for two weeks without any end-to-end demo. Use when a design doc proposes building the database layer first, then the API, then the UI — in sequence. Use when a teammate says "we'll integrate it all at the end." Use when the first user-visible result is many sprints away.
---

# Walking Skeleton

## Overview

**Build the thinnest possible end-to-end slice first** — UI → handler → domain → data → response → UI — and make sure it walks. Then thicken each layer.

A walking skeleton has *bones in every layer* and *muscle in none*. The point is the connections — every wire, every type contract, every deploy seam — proven end-to-end before any layer accumulates depth.

## When to Use

- Starting a new feature or new project
- Tempted to build "the data layer, then the API, then the UI" in sequence
- A design doc has detailed plans for layers that don't yet connect
- The integration step is on the roadmap as a separate milestone
- A new framework or library is being adopted across all features in parallel

## The Iron Rule

```
NEVER perfect one layer before connecting all of them. Joints first; muscle second.
```

**No exceptions:**
- Not for "the DB needs to be designed first"
- Not for "we can't deploy yet — it's not finished"
- Not for "building it end-to-end is more work"
- Not for "the skeleton is throwaway code"

## Detection: The Vertical-Tower Smell

If the plan has sequential layer milestones with no end-to-end demo until the last one, STOP and rework into a thin slice first:

```
❌ VERTICAL TOWER plan:
  Sprint 1: design and build the schema
  Sprint 2: build the API endpoints
  Sprint 3: build the UI
  Sprint 4: integration + deploy

✅ WALKING SKELETON plan:
  Day 1:   thin slice — stub handler returns hardcoded shape, page calls it,
           result renders. Deployed.
  Day 2+:  thicken: real input parsing
  Day 3+:  thicken: real persistence
  Day 4+:  thicken: real domain logic
  ...      each step is deployed, demonstrable, and small.
```

## Why It Wins

| Vertical tower | Walking skeleton |
|---|---|
| Integration cost lands at the end, all at once | Integration cost spread across the build, small per increment |
| Type contracts collide when layers finally meet | Type contracts pinned on day 1 |
| Deploy quirks discovered pre-launch | Deploy quirks discovered every PR |
| Stakeholders see no motion | Stakeholders see a working demo from day 1 |
| Architecture surprises late, expensive to fix | Architecture surprises early, cheap to fix |
| First user-visible result: weeks | First user-visible result: hours |

## The Pattern

### Day 1 — the absolute minimum that walks

For a new feature, build the thinnest stack that produces a user-visible result:

```ts
// handler — exists, returns a hardcoded shape
export async function handlePostCheckout(req: Request): Promise<Response> {
  return Response.json({ orderId: 'stub-order-1' });
}

// page — exists, renders a form, calls the handler
export default function CheckoutPage() {
  return (
    <form action={placeOrder}>
      <button type="submit">Place order</button>
    </form>
  );
}

// action — exists, calls the handler, returns to the user
export async function placeOrder() {
  const res = await fetch('/api/checkout', { method: 'POST' });
  const { orderId } = await res.json();
  return { orderId };
}
```

That's the skeleton. It walks: user clicks → action runs → handler runs → response returns → user sees a result. No domain logic, no DB, no validation, no styling. Every joint is wired and proven. The deploy works. The error path is visible — break the handler, and you see a real error in the UI, not a black box.

### From here, every PR thickens one layer

Each PR is small, deployable, and end-to-end:

- Add schema validation at the handler boundary; UI shows validation errors.
- Add the DB insert; handler returns a real `orderId`.
- Add the payment call; handler returns success/failure.
- Add idempotency keys.
- Add observability tags.
- Add timeouts and retries.

At every stage, the feature *walks* — even if the muscles are still skeletal.

### What's on the critical path *first*

The skeleton must touch every layer that will be on the critical path:

1. UI render
2. User action (form submit, button click)
3. Parse / validation at the boundary (even if trivial)
4. Domain operation (a real function, however thin)
5. Persistence (even of a stub row)
6. Response shape (typed)
7. Render the result

Skip any of these on day 1 and you skip the joint that will actually surprise you.

### What's *not* on day 1

- **Auth.** Stub or skip; revisit when the skeleton walks.
- **Styling.** Plain HTML is fine.
- **Error handling beyond "it errors."** The error path *is* on day 1; refined error messages are not.
- **Observability.** Add when the skeleton walks; `console.log` is enough until then.
- **Performance.** Measure first; optimize when measurements show a problem.
- **Edge cases.** The skeleton handles the happy path. Edge cases are muscle.

### Walking skeleton vs MVP

They're related but not the same:

- An **MVP** is the smallest *user-valuable* product — the thinnest feature set a real user would pay for.
- A **walking skeleton** is the smallest *technically-end-to-end* slice — even if no real user would care about it yet.

You build the skeleton *inside* the MVP. The MVP defines the feature's outer envelope; the skeleton is the first internal slice that connects every layer.

### The skeleton is deployed

The walking skeleton lives in production from day 1, behind a flag or at an internal URL. The deploy isn't ceremonial — it's the *test that the joints really hold*. Local dev hides plenty of deploy-time issues (env vars, build cache, runtime mismatches). The deploy is the proof.

## Pressure Resistance Protocol

### 1. "The DB needs to be designed first"

**Pressure:** "We can't build anything without knowing the schema."

**Response:** The schema *will* change as the feature evolves. Designing it in detail before any code uses it is speculative. Ship the skeleton with a minimal table (`id`, `createdAt`, two obvious columns); iterate as the feature reveals what it actually needs.

**Action:** Start the skeleton with a 3-column table. Add columns as the muscle grows.

### 2. "We can't deploy yet — it's not finished"

**Pressure:** "Deploying half-done code looks unprofessional."

**Response:** You're not deploying the finished feature; you're deploying the skeleton. It's labeled in-progress (a flag, a feature gate, or an internal URL). The deploy isn't the announcement; it's the *proof* the joints work.

**Action:** Ship behind a flag. Test the deploy machinery itself.

### 3. "Building it end-to-end is more total work"

**Pressure:** "Doing each layer once is more efficient than doing thin slices."

**Response:** It's *less* total work. The vertical-tower approach pays the integration cost in one painful late-stage chunk that often surfaces architectural rework. The skeleton spreads integration across the build and makes it small per increment.

**Action:** Build the joints first; the muscles fit into them naturally afterward.

### 4. "I don't know what the API should look like yet"

**Pressure:** "I need to figure out the design before I can implement."

**Response:** That's the *reason* to build the skeleton. The API shape gets pinned by the act of wiring it. A skeleton that walks tells you what the API actually needs; design-first guesses at what the API *might* need.

**Action:** Stub the API shape. Refine after the joints prove.

### 5. "The skeleton is too trivial to be useful"

**Pressure:** "Returning a hardcoded order ID doesn't prove anything."

**Response:** The skeleton's *value* is exactly that it's trivial. The triviality is what makes the joints visible. A skeleton that's already complicated has hidden some of the joints in the complication.

**Action:** Keep the body trivial. Trust the joints to be the lesson.

## Red Flags — STOP and Reconsider

- A project plan with sequential layer-by-layer milestones
- A new feature with no deployed preview after 2 weeks
- A design doc that specifies all the layers in detail but doesn't say how they connect
- A PR that adds a layer's caching / auth / observability before any end-to-end user flow exists
- A teammate's branch has thousands of lines of "foundation" with no working demo
- The phrase "integration is at the end" in a project plan
- A new ORM / framework / library being introduced across all features in parallel, with no single feature working in the new world
- Code coverage is 90% on one layer, 0% on others

**All of these mean: the joints are deferred — collapse the plan into a thin slice.**

## Quick Reference

| Situation | Action |
|---|---|
| New feature, day 1 | Build the thinnest end-to-end slice; deploy it |
| Plan proposes sequential layers | Replace with horizontal thin slices, deployed every PR |
| "Let me get the DB right first" | Stub the DB with a 3-column table; iterate later |
| "Can't deploy yet" | Deploy behind a flag; the deploy *is* the test |
| 2+ weeks with no demo | Drop everything; ship the skeleton |

## Common Rationalizations (All Invalid)

| Excuse | Reality |
|---|---|
| "Building one layer at a time is more focused" | Focused on the wrong axis. The hard problems are between layers. |
| "We'll integrate at the end" | The end is where integration is most expensive. |
| "The skeleton is throwaway code" | Skeleton becomes the file structure and type contracts. Mostly survives into the final feature. |
| "We need to figure out the design first" | The design *emerges* from the skeleton walking. Design-first guesses. |
| "Deploying half-done code is unprofessional" | Deploying behind a flag is a normal industrial practice. |
| "It's just a small project" | Small projects benefit *more* — integration cost is a higher fraction of total. |
| "The team is too small for this discipline" | Discipline scales *down*; small teams need it more. |

## The Bottom Line

**Walking skeleton on day 1. Muscle later. Joints prove first.**

The first commit that "works" is the one where a user-visible action travels the full stack and produces a user-visible result, however trivial. Everything after is thickening.

## Reference

- Alistair Cockburn, *Crystal Clear* (2004) — introduced "walking skeleton" as a methodology element. *"A walking skeleton is a tiny implementation of the system that performs a small end-to-end function. It need not use the final architecture, but it should link together the main architectural components."*
- Steve Freeman & Nat Pryce, *Growing Object-Oriented Software, Guided by Tests* (2009), ch. 4 — the most concrete operational treatment, with a worked example.
- Andy Hunt & Dave Thomas, *The Pragmatic Programmer* 20th-anniversary edition (2019), Tip 20 ("Use Tracer Bullets") — same idea, different name. *"Tracer bullets work because they operate in the same environment and under the same constraints as the real bullets."*
