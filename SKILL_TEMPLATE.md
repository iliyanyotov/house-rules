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
## When to Use       — 3-5 concrete triggers
## The Iron Rule     — code-fence "NEVER ..." + No exceptions list
## Detection: The "<X>" Smell — one before/after example
## Why <This> Wins   — 2-column table
## Pressure Resistance — 3-5 entries, each "Pressure / Response / Action"
## Red Flags         — bullet list
## Quick Reference   — situation → action table
## Common Rationalizations — excuse → reality table
## The Bottom Line   — one punch line + 2-sentence closer
## Related           — genuine neighbors, backticked names + reason (omit if none)
## Reference         — 2-4 citations
```

### Kind 2 — Pattern reference (150–200 lines)

For rules whose value is the pattern itself: naming conventions, type-design conventions, structural rules. Examples: `naming-ahclc`, `branded-ids`.

```
## Overview
## The Iron Rule
## The Pattern: <name>
### <Sub-rule 1>  — tables or short examples
### <Sub-rule 2>
### <Sub-rule 3>
## Worked Examples  — one consolidated block of short, real-looking examples
## Pressure Resistance  — bold prose, not numbered (saves space)
## Common Mistakes  — one consolidated smell → fix table
## The Bottom Line
## Related           — genuine neighbors, backticked names + reason (omit if none)
## Reference
```

### Kind 3 — Technical rule with multiple worked patterns (180–240 lines)

For skills where the rule needs several sub-patterns to be useful: resilience, type-driven, testing. Examples: `parse-dont-validate`, `graceful-degradation-defaults`, `timeouts-everywhere`.

```
## Overview
## The Iron Rule
## Why              — 1-2 paragraphs on the concrete failure mode
## Detection        — bullet list of signals
## The Pattern
### <Sub-pattern 1>  — one short before/after example
### <Sub-pattern 2>
### <Sub-pattern 3>
## Pressure Resistance
## Red Flags
## Common Rationalizations
## Related           — genuine neighbors, backticked names + reason (omit if none)
## Reference
```

## Conventions across all kinds

- **Generic examples.** No proper nouns from real apps, no domain entities from a real codebase. Use universal domains: `User`, `Invoice`, `Order`, `Subscription`.
- **TypeScript-family is fine.** Plain TS, plain Node, plain `async`/`fetch`. Mention frameworks (Next, Express, React) as one-line annotations, never bake an example into one framework's mental model.
- **One excellent example beats many.** If you can show the pattern once, do.
- **Directive tone.** "Never use a name that needs surrounding code." Not "consider whether…"
- **No decorative emoji.** ❌ / ✅ markers in before/after code comparisons are allowed (and standard in Anthropic's own skills); anything else (🎉 ✨ 👉) is not.
- **Cross-reference related skills by name, in prose.** Claude Code resolves no clickable link between skills — markdown file links break at runtime and `[[wiki]]` syntax is inert. The only thing that works is a plain backticked name (`` `fail-fast` ``). Where a skill has genuine neighbors (a tension it must reconcile, a handoff, a discipline it builds on), close the body with a `## Related` line listing them: backticked names + a few words on the relationship. Keep each skill self-contained — the line points the reader onward, it does not assume the neighbor was read.

## Verification checklist

- [ ] Frontmatter `name` is kebab-case; `description` lists triggers only (no workflow summary)
- [ ] Body fits Kind 1, 2, or 3 — or a justified hybrid
- [ ] Length is in the target range for its kind
- [ ] No project-specific proper nouns or domain entities lifted from real code
- [ ] Examples use universal-sounding domains (`User`, `Invoice`, `Order`)
- [ ] Not baked into one framework's mental model
- [ ] Iron Rule, at least one example, Pressure Resistance, References — all present
- [ ] `## Related` lists genuine neighbors by backticked name (or is omitted if the skill stands alone) — no `[[wiki]]` or markdown file links, which don't resolve
- [ ] No decorative emoji (❌/✅ comparators are allowed)

## Reference

- [Anthropic's `writing-skills` skill](https://github.com/anthropics) — the canonical guidance on skill authoring.
- [agentskills.io/specification](https://agentskills.io/specification) — the Agent Skills specification.
