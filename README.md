# House Rules

**Claude Code skills for writing software that holds up.**

Each skill is a self-contained rule with a worked example and citations. Each one comes from the same four places: the books and talks worth re-reading, the bugs that woke someone at 3am and were still memorable a year later, the refactors of code nobody planned to inherit, and the patterns that held no matter the team shape — solo, small, large, distributed, in-person, fast-iteration, or long-running product.

## Install

In Claude Code, add this repo as a plugin marketplace, then install the plugin:

```
/plugin marketplace add iliyanyotov/house-rules
/plugin install house-rules@house-rules
```

The skills load automatically and fire on relevance from then on. To update, run `/plugin marketplace update house-rules`.

## What's in it

Type-driven correctness, module shape, resilience, testing discipline, security (authorization, injection, supply chain), observability (structured logging, health checks, metric cardinality), async correctness, data and migrations (expand/contract, transaction isolation, N+1, the outbox pattern), change management, and the meta-principles (KISS, YAGNI, naming). Examples lean TypeScript / Bun / Node.js / Postgres; the rules are language-agnostic.

Each rule is labeled at its honest strength — an **Iron Rule** (a genuine invariant), **the Default Rule** (right unless named exceptions apply), or **a Heuristic** (a smell worth investigating) — so a rule you can't actually hold everywhere doesn't pretend to be one.

## Running the whole skillset on a project

If you want to run the whole skillset on a project — instead of waiting for each skill to fire on its own — paste this into Claude Code from inside the repo:

> Audit this codebase against every **house-rules** skill. Read each skill before judging, not just its name. Group findings by area (security, resilience, data, types, async, testing, observability, structure) and within each area sort by severity. For each finding give the file, the skill it breaks, the severity, and the smallest fix. Report first — don't change anything yet.

Scope a large repo or monorepo by naming a path (*"just `packages/api/`"*) or a category (*"just the resilience rules"*).

## How a skill works

Each skill is one `SKILL.md`:

```yaml
---
name: parse-dont-validate
description: Use when accepting external input. Use when tempted to call `z.parse()` more than once in a request path.
---

# Parse, don't validate
## The iron rule
**At every system boundary, transform `unknown` into a narrowed, branded type — once.**
...
```

Claude Code indexes the `description:` of every installed skill and loads the matching one into context when your work fits the trigger. Skills fire on relevance — or you invoke one by name: *"use `parse-dont-validate` on this handler."*

## Writing or fixing one

Every skill follows [`SKILL_TEMPLATE.md`](./SKILL_TEMPLATE.md): trigger-only frontmatter, a headline rule labeled at its honest strength (`## The Iron Rule`, `## The Default Rule`, or `## The Heuristic`), generic examples, and 2–4 citations. Run new skills through the checklist at the bottom of the template before opening a PR.

Bar for changes: show me the code where this rule would have prevented the bug.

MIT — see [LICENSE](LICENSE).
