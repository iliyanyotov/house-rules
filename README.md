# House Rules

Claude Code skills for well-made software. Language-agnostic discipline, TypeScript-flavored craft.

> ⚠️ Work in progress. Skills are still being authored — names and scope may still shift.

## Why

Most engineering "best practices" are either too abstract to act on or too tied to one framework to survive a stack change. AI pair-programmers make the gap worse — a model that hasn't been told *the boundary parses, the interior trusts* will sprinkle `z.parse()` through four layers.

This repo is the explicit version: small, sharp, citation-heavy rules that fire only when their trigger matches what Claude is doing.

## What's in it

Type-driven correctness, module shape, resilience, testing discipline, change management, and the meta-principles (KISS, YAGNI, naming). Examples lean TypeScript / Next.js / Drizzle / Vercel; the rules are language-agnostic. Current index: [`skills/`](./skills/).

## How a skill works

Each skill is one `SKILL.md` with two parts:

```yaml
---
name: parse-dont-validate
description: Use when accepting external input. Use when tempted to call `z.parse()` more than once in a request path.
---

# Parse, Don't Validate
## The Iron Rule
**At every system boundary, transform `unknown` into a narrowed, branded type — once.**
...
```

Claude Code indexes the `description:` from every skill. When your work matches one of those triggers, the skill loads into context and Claude follows it. **You don't invoke skills manually** — they fire on relevance. You can also call one by name: *"use `parse-dont-validate` on this handler."*

## Writing or fixing one

Every skill follows [`SKILL_TEMPLATE.md`](./SKILL_TEMPLATE.md): trigger-only frontmatter, one of three body shapes (posture / pattern reference / technical rule), generic examples (no real project names), and 2–4 citations. Run new skills through the checklist at the bottom of the template before opening a PR.

The bar for changes: *"show me the code where this rule would have prevented the bug."*

## Status

Authored from a curated reading list ([`docs/skill-source-inventory.md`](docs/skill-source-inventory.md)) and validated against real projects ([`docs/validations/`](docs/validations/)) before each release.

MIT — see [LICENSE](LICENSE).
