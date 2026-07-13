# SKILL_TEMPLATE.md

How to structure a `SKILL.md` in this library. Follows Anthropic's conventions from the official `writing-skills` skill.

## Frontmatter

```yaml
---
name: kebab-case-name
description: Use when <trigger>. Use when <trigger>. ...
---
```

- `name` — letters, numbers, hyphens. Matches the directory.
- `description` — Claude reads this to decide whether to load the skill. **Describe *when to use*, not what the skill does.** Stack concrete "Use when…" triggers. Never summarize the workflow — descriptions that summarize cause Claude to skip the body.

```yaml
# ❌ Summarizes the workflow
description: Use for TDD — write test first, watch it fail, write minimal code, refactor

# ✅ Triggers only
description: Use when implementing any new feature or function. Use when asked to "add tests later". Use when "this is hard to test" comes up.
```

## Body — pick the kind that fits

Three shapes cover most skills. Don't force a template that doesn't fit.

### Kind 1 — Universal posture (130–180 lines)

For rules that fire on every line or every PR. Examples: `tdd`, `kiss`, `yagni`, `small-changesets`, `dry`.

```
## Overview          — 2 lines: punch + supporting
## When to use       — 3-5 concrete triggers
## The iron rule     — headline rule; label at its true strength (see below):
##                     `## The iron rule` (invariant, code-fence "NEVER ..." + No exceptions),
##                     `## The default rule` (default, "Prefer ..." + "When this doesn't apply"),
##                     or `## The heuristic` (smell, "Treat ... as a signal" + "What doesn't trip it")
## Detection: the "<x>" smell — one before/after example
## Why <this> wins   — 2-column table
## Pressure resistance — 3-5 entries, each "Pressure / Response / Action"
## Red flags         — bullet list
## Quick reference   — situation → action table
## Common rationalizations — excuse → reality table
## The bottom line   — one punch line + 2-sentence closer
## Related           — genuine neighbors, backticked names + reason (omit if none)
## Reference         — 2-4 citations
```

### Kind 2 — Pattern reference (150–200 lines)

For rules whose value is the pattern itself: naming conventions, type-design conventions, structural rules. Examples: `naming-ahclc`, `branded-ids`.

```
## Overview
## The iron rule / default rule / heuristic   — headline rule at its true strength
## The pattern: <name>
### <Sub-rule 1>  — tables or short examples
### <Sub-rule 2>
### <Sub-rule 3>
## Worked examples  — one consolidated block of short, real-looking examples
## Pressure resistance  — bold prose, not numbered (saves space)
## Common mistakes  — one consolidated smell → fix table
## The bottom line
## Related           — genuine neighbors, backticked names + reason (omit if none)
## Reference
```

### Kind 3 — Technical rule with multiple worked patterns (180–240 lines)

For skills where the rule needs several sub-patterns to be useful: resilience, type-driven, testing. Examples: `parse-dont-validate`, `graceful-degradation-defaults`, `timeouts-everywhere`.

```
## Overview
## The iron rule / default rule / heuristic   — headline rule at its true strength
## Why              — 1-2 paragraphs on the concrete failure mode
## Detection        — bullet list of signals
## The pattern
### <Sub-pattern 1>  — one short before/after example
### <Sub-pattern 2>
### <Sub-pattern 3>
## Pressure resistance
## Red flags
## Common rationalizations
## Related           — genuine neighbors, backticked names + reason (omit if none)
## Reference
```

## Conventions across all kinds

- **Generic examples.** No proper nouns from real apps, no domain entities from a real codebase. Use universal domains: `User`, `Invoice`, `Order`, `Subscription`.
- **TypeScript-family is fine.** Plain TS, plain Node, plain `async`/`fetch`. Mention frameworks (Next, Express, React) as one-line annotations, never bake an example into one framework's mental model.
- **Assume a strict tsconfig.** Examples must type-check under `strict: true` **and** `noUncheckedIndexedAccess: true`. An index/`.find()`/`findFirst()` access yields `T | undefined` — handle it. Don't write an example that only compiles under looser flags.
- **One excellent example beats many.** If you can show the pattern once, do.
- **Directive tone.** "Never use a name that needs surrounding code." Not "consider whether…"
- **No decorative emoji.** ❌ / ✅ markers in before/after code comparisons are allowed (and standard in Anthropic's own skills); anything else (🎉 ✨ 👉) is not.
- **Cross-reference related skills by name, in prose.** Claude Code resolves no clickable link between skills — markdown file links break at runtime and `[[wiki]]` syntax is inert. The only thing that works is a plain backticked name (`` `fail-fast` ``). Where a skill has genuine neighbors (a tension it must reconcile, a handoff, a discipline it builds on), close the body with a `## Related` line listing them: backticked names + a few words on the relationship. Keep each skill self-contained — the line points the reader onward, it does not assume the neighbor was read.
- **High-signal triggers only.** A `description:` fires the skill automatically, so its "Use when…" triggers must be *failure-shaped and specific* ("Use when a `switch` over a union has no `default`"), not broad postures ("Use when writing code"). Broad triggers collide — many skills load on one ordinary task and drown the signal. If a skill is meant to be *reached for deliberately* rather than auto-firing, say so in its first trigger ("Use when explicitly reviewing X").

## The core rule: invariant, default, or heuristic

A skill's headline rule sits at one of three strengths. Label it honestly — don't inflate a default into a `NEVER`, or the body's own carve-outs will contradict the headline.

- **Invariant** — genuinely universal *within the stated scope*. Only these earn a code-fenced `NEVER …` plus a real "No exceptions" list. (E.g. "at a boundary, turn `unknown` into a narrowed type once.")
- **Default** — the right call unless *named* conditions apply. Write it as "Prefer X; the exceptions are A, B, C" and then actually list them. Don't write `NEVER … / No exceptions` and then spend a later section on exceptions.
- **Heuristic** — a diagnostic signal needing judgment ("a `switch` with 8+ cases is a smell"). Frame it as a smell to investigate, not a law.

Test: if the body later says "except when…", the headline was a *default*, not an invariant — reword the headline, or fold the case into the No-exceptions list as a genuine non-exception. A rule you have to walk back isn't an Iron Rule.

## When skills conflict — the priority order

Many skills fire on one task; when their advice pulls in different directions, resolve in this order (higher wins):

1. **Correctness & security** — data integrity, auth, idempotency, no secret/PII leaks. Never traded for style.
2. **Explicit product contract** — a stated requirement or API guarantee.
3. **Repository convention** — how *this* codebase already does it.
4. **Heuristic style guidance** — the posture skills (KISS, YAGNI, naming, DRY).

So `yagni` yields to a resilience skill on a real safety control; a naming heuristic yields to an established repo convention. Where two skills carry a standing tension, name the reconciliation in each one's `## Related` line so the reader sees the boundary.

## Verification checklist

- [ ] Frontmatter `name` is kebab-case; `description` lists triggers only (no workflow summary)
- [ ] Body fits Kind 1, 2, or 3 — or a justified hybrid
- [ ] Length is in the target range for its kind
- [ ] No project-specific proper nouns or domain entities lifted from real code
- [ ] Examples use universal-sounding domains (`User`, `Invoice`, `Order`)
- [ ] Not baked into one framework's mental model
- [ ] Headline uses the heading that matches its true strength — `## The Iron Rule` (invariant), `## The Default Rule` (default), or `## The Heuristic` (heuristic); at least one example; Pressure Resistance; References — all present
- [ ] `## Related` lists genuine neighbors by backticked name (or is omitted if the skill stands alone) — no `[[wiki]]` or markdown file links, which don't resolve
- [ ] No decorative emoji (❌/✅ comparators are allowed)
- [ ] `description` triggers are failure-shaped and specific, not broad postures (high-signal, so the skill doesn't collide on every task)
- [ ] Headline rule is labeled at its true strength — a `## The Iron Rule` with `NEVER … / No exceptions` contains no case the body later walks back (else make it `## The Default Rule` with a "When this doesn't apply" list, or `## The Heuristic` with a "What doesn't trip it" list)

## Reference

- [Anthropic's skills repository](https://github.com/anthropics/skills) — the canonical guidance and examples for skill authoring.
- [agentskills.io/specification](https://agentskills.io/specification) — the Agent Skills specification.
