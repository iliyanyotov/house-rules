# House Rules

**Claude Code skills for writing software that holds up.**

> **Status: unreleased and not yet properly versioned.** This plugin is still being evaluated against a range of real codebases; treat it as a work in progress. It carries no meaningful version yet, and skills may be added, reworded, or removed without notice until it settles.

Every skill comes from the same four places: the books and talks worth re-reading, the bugs that woke someone up at 3am and stayed memorable a year later, the refactors of code nobody planned to inherit, and the patterns that kept showing up no matter how the team was shaped — solo, small, large, distributed, in-person, fast-iteration, or a long-running product.

## What's in it

Type-driven correctness, module shape, resilience, testing discipline, change management, and the meta-principles (KISS, YAGNI, naming). Examples lean TypeScript / Node.js / Postgres; the rules themselves are language-agnostic.

## How a skill works

Each skill is one `SKILL.md`:

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

Claude Code indexes the `description:` from every installed skill and loads the matching one into context when your work fits the trigger. Skills fire on relevance; you can also invoke one by name: *"use `parse-dont-validate` on this handler."*

## Writing or fixing one

Every skill follows [`SKILL_TEMPLATE.md`](./SKILL_TEMPLATE.md): trigger-only frontmatter, one of three body shapes (posture, pattern reference, or technical rule), generic examples, 2–4 citations. Run new skills through the checklist at the bottom of the template before opening a PR.

Bar for changes: show me the code where this rule would have prevented the bug.

MIT — see [LICENSE](LICENSE).
