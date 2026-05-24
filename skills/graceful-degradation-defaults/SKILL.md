---
name: graceful-degradation-defaults
description: Use when designing a feature that depends on a non-critical external system (recommendations, search, social feed, analytics widget, third-party embed, AI generation). Use when a leaf-feature failure should never propagate as a 5xx to the user. Use when an incident report shows a full-page outage caused by one slow dependency. Use when adding a new optional capability to an existing page.
---

# Graceful Degradation Defaults

## Overview

**Non-critical features fail to a *named*, *documented* fallback** — never to a 500 or a blank page. The user-visible default for a dependency outage is defined at design time, not improvised at incident time.

If a feature is "non-critical," its failure produces a smaller-but-working page. If a feature is "critical," it's subject to higher availability requirements — not to this rule.

## The Iron Rule

```
NEVER let a non-critical feature's failure produce a 5xx or a blank space. Always fall back to a named state.
```

**No exceptions:**
- Not for "the user needs to know"
- Not for "stale data is worse than no data"
- Not for "we'll improvise the fallback if it ever breaks"
- Not for "the framework should handle this"

## Why

Most pages have a *core path* (sign in, see your data, take the primary action) and a *garnish* (recommendations, social feed, AI summary, related items). Garnish helps; it's not load-bearing.

When the garnish fails, the page should still render. The user should still complete the core path. The garnish's failure should be visible enough for operators to fix and invisible enough that users don't notice unless they were specifically there for the garnish.

Without this discipline, every garnish becomes a single point of failure for the whole page. A flaky recommendation service broken 1% of the time produces a 1% page-error rate — even though the core path works perfectly.

## Detection

You are violating the rule if any of these are true:

- A request handler returns 500 because one of N parallel fetches failed.
- A `Promise.all` across services of mixed criticality means *any* failure fails the lot.
- A non-critical fetch is `await`ed inline in the critical path with no fallback.
- A code review where "what does the user see if X is down?" has no answer.
- A loading state is also the failure state — the page just shows "loading…" forever.
- A third-party widget blocks the page render until it loads or times out.
- A retrospective: "the recommendations outage took down the whole dashboard."

## The Pattern

### Split criticality at the data layer

The rule lives in how you fan out to dependencies. Critical paths must succeed; garnish paths may fail and produce `null`.

```ts
async function fanOut(userId: UserId) {
  const [userR, ordersR, recsR] = await Promise.allSettled([
    fetchUser(userId),             // critical
    fetchOrders(userId),           // critical
    fetchRecommendations(userId),  // garnish
  ]);

  // Critical paths — if either failed, the page can't render.
  if (userR.status === 'rejected') throw userR.reason;
  if (ordersR.status === 'rejected') throw ordersR.reason;

  // Garnish — log and fall through with null.
  if (recsR.status === 'rejected') {
    log.warn('recommendations_unavailable', { userId, error: recsR.reason });
  }

  return {
    user: userR.value,
    orders: ordersR.value,
    recommendations: recsR.status === 'fulfilled' ? recsR.value : null,
  };
}
```

`Promise.allSettled` replaces `Promise.all` whenever the leaves have **different criticality**. The split is explicit. Critical results throw; garnish results return `null`. Consumers can tell the difference from the return type alone.

### The fallback is a named, honest state — not silence

When `recommendations` is `null`, the renderer shows a *named* fallback. Three rules:

- **Name** the missing thing.
- **Reassure** about what's still working.
- **Don't lie** — "loading…" forever is a lie if you've given up.
- **Be small** — a fallback that itself can fail is no fallback.

```ts
function renderRecommendations(recs: Recommendation[] | null): string {
  if (recs === null) {
    return `<section aria-label="Recommendations">
      Recommendations are temporarily unavailable. The rest of your dashboard is up to date.
    </section>`;
  }
  if (recs.length === 0) {
    return `<section>No recommendations yet. Check back tomorrow.</section>`;
  }
  return renderList(recs);
}
```

Three distinct states (broken, empty, loaded), three distinct visuals. The type `Recommendation[] | null` already encodes "broken or loaded"; empty is just `length === 0` on a loaded result.

### Stale-cache fallback — for occasionally-available data

When the dependency is sometimes available, falling back to recently-fresh data beats falling back to nothing:

```ts
async function fetchRecommendations(userId: UserId): Promise<Recommendation[]> {
  try {
    const fresh = await api.recommendations(userId, { signal: AbortSignal.timeout(2_000) });
    await cache.set(`recs:${userId}`, fresh, { ttlSeconds: 86_400 });
    return fresh;
  } catch (err) {
    const stale = await cache.get<Recommendation[]>(`recs:${userId}`);
    if (stale) {
      log.warn('recommendations_stale_fallback', { userId });
      return stale;
    }
    throw err; // bubble up; caller renders the "unavailable" fallback
  }
}
```

Watch the cardinality of the cache key. Keyed by `userId` is fine for bounded user counts; higher-cardinality keys need a TTL strategy.

### Framework binding — same rule, different attachment

The rule is the same; the *binding* differs.

- **Next.js (App Router):** wrap each garnish region in a `<Suspense>` (slow) and an `<ErrorBoundary>` (failed). The boundary's `fallback` prop renders the named fallback component.
- **Express / Fastify / Koa:** the handler builds the page model from the `fanOut` above; the template renders the named fallback when fields are `null`.
- **A worker or job:** the job records the garnish failure as a warning, completes its critical work, and ships.

In every case the *split* happens at the data layer (`Promise.allSettled` + criticality labels), and the *fallback* is a named component or template branch — not a thrown error, not a blank space, not infinite loading.

### What is "critical" vs "garnish"

Decision-time question: *"Should this feature's outage block the user from doing what they came here to do?"*

| Page | Critical | Garnish |
|---|---|---|
| Sign-in | Auth | Welcome message, marketing tile |
| Dashboard | Core stats | Recommendations, related links |
| Checkout | Payment processor | Address autocomplete, gift-message |
| Article view | Article content | Related-stories panel, comments load |
| Settings | Save action | Avatar preview, social-link fetch |

When in doubt: *if this fails forever, can the product still ship?* If yes, garnish.

## Pressure Resistance

### "Showing an error is honest, fallbacks hide problems"

Fallbacks aren't silent — they emit logs and metrics; operators see them. What they *don't* do is take an unrelated workflow down. "Honest to the user" isn't "broken for the user."

### "Stale data is worse than no data"

Sometimes — for fast-moving data (stock prices, live scores). For most app data (recommendations, profile snippets, related items), stale is dramatically better than empty. Decide per-feature; default to "stale OK."

### "We don't have time to design fallbacks for every feature"

Then you'll improvise them under incident pressure, which is the wrong time to design UX. Spend an hour during feature design; save days during incidents.

### "Promise.all is simpler than allSettled"

It's also less correct when the leaves have different criticality. The simplification produces a system where any leaf failure is a root failure.

### "If the service is down, the team needs to know"

They will — from logs, metrics, breaker state. They don't need to know via *every user's broken page*. Operator-visibility and user-impact are different needs.

### "I'll add a try/catch around it"

A bare `try { ... } catch { return null; }` collapses three states (loading, broken, empty) into one. The caller can't tell what happened. Push the criticality decision into the *fan-out shape*, not into ad-hoc try/catch.

## Red Flags

- A `Promise.all` of mixed-criticality fetches.
- A blank section with no fallback when its data is missing.
- A loading skeleton that's also the failure state.
- A `try { ... } catch { return null; }` with no log, no metric, no named state.
- A code review where "what do users see during incident" hasn't been answered.
- A retrospective: "the X outage took down the Y."
- The phrase "this should never fail" applied to anything external.

**All of these mean: criticality isn't split — name what's garnish and let it fail to a fallback.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's a small feature, not worth a fallback" | Small features cause big page outages without fallbacks. |
| "Failing loudly forces us to fix it" | It forces *users* to deal with it. Use observability for force, not user pain. |
| "We don't know what the right fallback is" | "Section unavailable; the rest of the page works" is always a valid fallback. |
| "Caching adds staleness bugs" | Bounded TTL (minutes to hours) for non-critical garnish is universally safe. |
| "The framework should handle this" | Frameworks give you the primitives. Wiring them is your job. |
| "We have a global 500 page, that's the fallback" | A global 500 turns a leaf failure into a full-page failure. The opposite of graceful. |

## Reference

- Michael Nygard, *Release It!* 2e (2018) — *steady state* and *fail fast* patterns. Stability is achieved by isolating failure domains, not by preventing all failures.
- Netflix Hystrix design notes — *fallback* as a first-class concept in resilience. The library is in maintenance, but the pattern is canonical.
- Google SRE Book, ch. 4 ("Service Level Objectives") — different features can have different SLOs; the page's overall SLO is *not* the AND of all feature SLOs unless you design it that way.
