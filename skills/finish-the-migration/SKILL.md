---
name: finish-the-migration
description: Use when introducing a replacement for an existing pattern, library, module, or API — a new data-access layer beside the old one, a v2 client, a new state container, a different validation approach. Use when "the new way" and "the old way" both exist and both are reachable. Use when a doc or CLAUDE.md says "use X for new code, don't use Y" but Y is still wired and used. Use when a migration has been "90% done" for months. Use when two implementations of one concept coexist with no removal date.
---

# Finish the Migration

## Overview

**When you introduce a replacement for an existing pattern, you take on a debt: the old path and the new path now both exist. That debt must be tracked, fenced, and scheduled for payoff — not left open-ended.** A migration isn't done when the new way works; it's done when the old way is *gone*.

This is the complement to `dead-code-deletion-on-sight`. That skill deletes code with **zero references** the moment you find it. This skill governs the harder case: an old path that is *still referenced* and still working, sitting beside its replacement. You can't just delete it — but you must not let it live forever either.

## The Iron Rule

```
NEVER leave a superseded path reachable without a deprecation marker, a guard against new uses, and a removal trigger.
```

**No exceptions:**
- Not "the new pattern is the standard now, everyone knows to use it"
- Not "a doc says don't use the old one" (a doc is not a guard)
- Not "we'll migrate the stragglers eventually"
- Not "both work, so there's no urgency"

## Why

Two implementations of one concept is the most expensive state a codebase can be in — more expensive than either the old way *or* the new way alone. Every reader must learn both. Every change risks being applied to the wrong one. New code copies whichever the author happened to see first, so the old path keeps *growing* even after it's "deprecated." Bugs get fixed in one and not the other. The two drift until they behave differently, and now the fork is load-bearing — too risky to remove because nobody's sure what depends on the old behavior.

"Use the new way for new code" is not a plan; it's a hope. Hope doesn't stop the next engineer — who never read the doc — from extending the old path. The only things that actually halt the bleed are a *machine-enforced* guard and a *scheduled* removal with an owner. A migration without a finish line isn't a migration; it's a permanent second way of doing everything.

## Detection

You are leaving a migration unfinished if any of these are true:

- A replacement exists, but the thing it replaced is still imported in new-ish code.
- The only thing stopping new uses of the old path is a sentence in a README/CLAUDE.md.
- You can't answer "how many call sites still use the old way?" with a number.
- You can't answer "when does the old way get deleted, and who owns that?"
- A bug was fixed in the new path but not the old (or vice versa).
- The phrase "we're mostly migrated" has been true for more than one release cycle.
- A code reviewer can't tell, from the code alone, which of two patterns is preferred.

## The Pattern

### Step 1 — Make the replacement the obvious default, then *mark* the old path

A doc is not a guard, but a deprecation marker in the code is a start — it shows up at every call site:

```ts
// ❌ The old path looks exactly as legitimate as the new one. New code will pick it.
export class UserModel { /* legacy data access */ }
export class UserRepository { /* preferred data access */ }

// ✅ The old path announces itself at every use site.
/** @deprecated Use UserRepository. Tracked for removal in #1234. */
export class UserModel { /* legacy data access */ }
export class UserRepository { /* preferred data access */ }
```

Most real `@deprecated` markers stop at "Use X instead" — that half is *advisory*, a nicer doc comment. The removal trigger (`#1234`, a date, or an owner) is the half that turns the marker into a *commitment*: it's what a stale-deprecation audit greps for. A `@deprecated` with no trigger is how migrations stall for years.

### Step 2 — Stop the bleed with a machine guard

The decisive move: make *new* uses of the old path fail CI. Whatever the rule is — "no new imports of `models/`", "no new calls to `legacyClient`" — encode it so a human doesn't have to catch it in review.

When the deprecated thing is a *member* of a module whose other members are still preferred — a `@deprecated` default export beside a good named export, one bad function in an otherwise-fine module — a path ban won't work: `import { good } from 'x'` must stay legal. Guard the *specific symbol* instead (restrict the default import, or the named symbol via `no-restricted-imports`'s `importNames`), not the whole module path.

```jsonc
// Example: an ESLint no-restricted-imports / no-restricted-syntax rule.
// The point is not the tool — it's that a NEW use of the old path breaks the build.
{
  "rules": {
    "no-restricted-imports": ["error", {
      "patterns": [{
        "group": ["**/models/*"],
        "message": "models/ is being retired — use the matching repositories/ class. See #1234."
      }]
    }]
  }
}
```

Now the count of old-path uses can only go *down*. Without this, every other step is undermined by new code re-growing the old path.

The mechanism is interchangeable: Biome (`style/noRestrictedImports`), a lint deprecation rule (`@typescript-eslint`'s `no-deprecated`, Biome's `noDeprecated` on a `@deprecated`-tagged symbol), or even a one-line grep check in CI all do the job. Pick whatever your stack already runs — the only requirement is that a *new* use of the old path fails the build, not that any specific tool enforces it.

### Step 3 — Track the remaining call sites as a countable, shrinking list

Put the finish line somewhere visible: a tracking issue with the current count, or a checklist of remaining sites. The number must trend to zero. A migration you can't measure is one you'll never finish.

```
# Migration: models/ → repositories/   (issue #1234)
Remaining call sites: 8 → 5 → 2 → 0
- [x] orderModel  → orderRepository
- [x] userModel   → userRepository
- [ ] bookingModel → bookingRepository   ← blocked on the relations query
- [ ] locationModel → locationRepository
Removal trigger: when count hits 0, delete models/ and the BaseModel base class in the same PR.
```

### Step 4 — Migrate incrementally, then delete the old path entirely

Convert call sites in small changesets (one entity, one module at a time — see `small-changesets`). When the count hits zero, the old path has **zero references** — and now `dead-code-deletion-on-sight` takes over: delete it, including its base classes, tests, and config. The migration is done only when the old code is *gone from the tree*, not when it's merely unused.

### The seam to `dead-code-deletion-on-sight`

The two skills hand off at the reference count:

| State of the old path | Skill that governs it |
|---|---|
| Still referenced, replacement exists | **finish-the-migration** — mark, guard, track, schedule |
| Zero references (count hit 0) | **dead-code-deletion-on-sight** — delete it now |

`finish-the-migration` exists to *drive the count to zero*; `dead-code-deletion-on-sight` exists to *act the instant it's there*.

## Pressure Resistance

**"The new way is the standard — everyone knows to use it."** Everyone who read the announcement, was on the team when it happened, and remembers. The next hire read none of that and will extend whichever path they grep into first. Knowledge in people's heads doesn't constrain a codebase; a CI guard does.

**"There's a doc saying don't use the old one."** A doc is advisory and invisible at the call site. It has never once stopped a busy engineer from importing the thing that's right there and compiles. If the rule matters, encode it as a build failure; if it doesn't matter enough to encode, it won't be followed.

**"We'll get to the stragglers eventually."** "Eventually" with no owner and no trigger means never — the stragglers become permanent, and meanwhile the guard you didn't add lets new ones appear. Set the removal trigger *when you start* the migration, not after. A migration without a finish line is just a second way of doing things, forever.

**"Both implementations work, so it's not urgent."** Both working is exactly the danger: nothing forces the issue, so the fork persists and the two drift until they *don't* behave the same. The cost isn't a broken build today; it's the slow tax of every reader learning both and every change risking the wrong one.

**"Deleting the old path is risky — things might depend on its quirks."** That risk *grows* the longer you wait, as more code piles onto the old path and its quirks calcify into contracts. The way to shrink the risk is to drive the reference count down deliberately now, while the surface is small — not to leave it and hope.

## Red Flags

- A replacement landed but the old path has no `@deprecated` marker.
- "Don't use X for new code" lives only in a doc, with nothing enforcing it.
- New commits still add references to the path that's supposedly being retired.
- No tracking issue, no call-site count, no removal date, no owner.
- A bug fixed in one of the two implementations but not the other.
- A base class / shared util kept alive solely for the old path.
- "Mostly migrated" repeated across multiple releases.

**All of these mean: the migration has no finish line — add the marker, the guard, the count, and the trigger.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Use the new way for new code" | Not a plan — a hope. Only a CI guard stops new uses of the old path. |
| "A doc already says don't use it" | Docs don't fail builds. The old path still compiles and gets imported. |
| "We'll migrate the rest later" | "Later" without an owner and a trigger is never. Set the trigger up front. |
| "Both work, no rush" | Both working is why it never gets finished — and why they drift apart. |
| "Deleting the old one is too risky" | The risk only compounds with time. Shrink it now while the surface is small. |
| "Keeping both is more flexible" | Two ways to do one thing is not flexibility; it's a permanent tax on every reader. |

## The Bottom Line

Introducing a replacement is signing up to *remove the original* — on a schedule, behind a guard, against a count that reaches zero. Until the old path is deleted from the tree, the migration isn't done; it's just two ways of doing one thing, costing you both.

## Related

- `dead-code-deletion-on-sight` — handoff: drive references to zero, then delete
- `decouple-deploy-from-release-via-flags` — migrate behind a flag, then remove it
- `expand-contract-schema-migration` — the schema-shaped instance of the same lifecycle

## Reference

- Martin Fowler, [*StranglerFigApplication*](https://martinfowler.com/bliki/StranglerFigApplication.html) — incremental replacement where the old system is progressively retired, not left running beside the new one indefinitely.
- Complements `dead-code-deletion-on-sight` (which removes zero-reference code on sight) and `small-changesets` (the unit of an incremental migration). The hand-off: drive references to zero, then delete.
