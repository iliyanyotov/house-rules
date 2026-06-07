---
name: no-barrel-files
description: Use when creating a new `index.ts` whose only purpose is to re-export from sibling files. Use when reviewing a `import { X } from '@/foo'` that resolves to `@/foo/index.ts` re-exporting `from './X'`. Use when Turbopack HMR feels slow, when the production bundle is larger than expected, or when "find references" on a symbol returns the barrel before the source. Use when adding shadcn/ui components or other libraries that ship barrel-shaped re-exports.
---

# No Barrel Files

## The Iron Rule

**Import from the file that defines the symbol. Never create an `index.ts` that exists only to re-export from sibling files.** Barrel files trade developer convenience (shorter imports) for build-system harm (defeated tree-shaking, slower HMR, broken "find references") and code-clarity harm (the path lies about where the symbol lives).

The one legitimate exception: a **public-API surface** for a package or feature module whose internals genuinely should be hidden. That's rare in app code and obvious when it applies.

## Why This Rule

A barrel file is `index.ts` whose body is:

```typescript
export { default as Foo } from './Foo';
export { default as Bar } from './Bar';
export { something } from './something';
```

It costs you:

- **Tree-shaking.** Bundlers can usually shake barrels, but Next.js's Turbopack (and webpack in dev mode) frequently can't statically determine that `import { Foo } from '@/components'` only needs `Foo` — so they pull in the *whole* barrel transitively. One unused import drags in dependencies of every re-exported sibling.
- **HMR cost.** Edit any sibling, and every consumer of the barrel re-evaluates. With component-level barrels, that's nearly every render.
- **Lying paths.** `import { Foo } from '@/components/foo'` resolves to `@/components/foo/index.ts` which lives next to `@/components/foo/Foo.tsx`. "Find definition" hops through the barrel; reading the file tells you nothing about where the symbol came from.
- **Circular-import bait.** A barrel that re-exports from `A` and `B` makes `A` and `B` *transitively* import each other through the barrel. Innocuous-looking code suddenly has cycles.
- **Stable-public-API illusion.** Barrels feel like "the public API of a feature." In a monorepo with versioned packages, that framing has merit. In a Next.js app, *the whole app is one package* — there's no consumer outside the codebase. The "public API" is whatever any file imports.

## Detection

You are violating the rule if any of these are true:

- An `index.ts` whose body is *only* `export { ... } from './...'` lines.
- A `re-export wrapper around `default`: `export { default } from './Foo';` — especially common for component-level barrels where the file name and the export name already match.
- An import path that ends in a directory name (`from '@/components/foo'`) when there's a same-named file inside it (`Foo.tsx`).
- A "find references" on a symbol returns the barrel before the source file.
- `bun run dev` HMR feels slower than expected after editing one component.
- The Next.js bundle analyzer shows components you don't import on a page anyway.

## The Correct Pattern

### Component-level barrels — almost always wrong

```typescript
// ❌ src/components/Weather/index.ts
export { default } from './Weather';
```

```typescript
// ❌ Consumer
import Weather from '@/components/Weather';
```

```typescript
// ✅ Delete the barrel.
// ✅ Consumer
import Weather from '@/components/Weather/Weather';
```

The shorter path was the *only* benefit of the barrel; the longer path is one extra `/Weather` and removes every other cost listed above.

### Feature-level public API — sometimes OK

When a feature has multiple related exports and the *list itself* is the contract, a barrel can earn its place:

```typescript
// 🟡 src/features/home/index.ts — feature's public surface
export { default as HomeClient } from './HomeClient/HomeClient';
export { default as Weather } from './Weather/Weather';
// Note: SpawningPeriods is *internal* — only HomeClient imports it, so it's not re-exported here.
```

```typescript
// Consumer in src/app/page.tsx
import { HomeClient, Weather } from '@/features/home';
```

The criterion: **the barrel lists the feature's intentional public surface, and at least one symbol from the feature is intentionally *not* re-exported (i.e., the barrel hides internals).** If the barrel re-exports everything, it's not curating — it's just a path-shortening tax.

Note the direct path inside the barrel: `from './HomeClient/HomeClient'` not `from './HomeClient'`. The barrel itself shouldn't import via sub-barrels — that compounds the problem.

### Library-installed barrels — known exception

shadcn/ui generates components into `src/components/ui/` with no barrel by default — which is correct. But the moment you write `import { Button, Card } from '@/components/ui'`, you've effectively created one. **Import direct from each component file.**

Same applies to icon libraries with named exports (`lucide-react`, `@tabler/icons-react`) — those are *packaged* barrels, controlled by the library author, and modern bundlers + the library's `"sideEffects": false` field cooperate to make tree-shaking work. Trust the library; don't re-wrap.

### When you have many small utility helpers

```typescript
// ❌ src/utils/index.ts re-exports all 8 helpers
export { default as formatDate } from './formatDate';
export { default as parseAmount } from './parseAmount';
// ... 6 more
```

```typescript
// Consumer
import { formatDate, parseAmount } from '@/utils';
```

Two paths forward:

1. **Direct imports** (preferred — no barrel needed):
   ```typescript
   import formatDate from '@/utils/formatDate';
   import parseAmount from '@/utils/parseAmount';
   ```
2. **Keep the barrel** *if* the utils module is genuinely the project's "standard library" with a curated surface. Be honest — most `utils/index.ts` files aren't curated.

## Pressure Resistance

### "Shorter imports are nicer"

Nicer to type, harder to navigate. The cost lands on every reader, every build, every refactor. The benefit lands on whoever typed the import once.

### "The bundler will tree-shake it"

Sometimes. With Next.js + Turbopack, *frequently not* — the dependency graph through a barrel is opaque enough that the bundler conservatively pulls in everything. The way to know is to run `next build` with `ANALYZE=true` and look. Most teams don't.

### "But shadcn/Storybook/X uses barrels"

shadcn doesn't generate barrels — it generates per-component files. Storybook's *stories* aren't barrels (they're auto-discovered by glob). The "everyone does it" is mostly third-party packages that *are* shipping curated public APIs — which is the one legitimate use case. Don't generalize from packaged libraries to your app code.

### "I'll have to update 20 imports if I remove the barrel"

Once. Then no more. The IDE's "Update Imports" refactor handles it. After that, every future import goes direct — no further cost. A 5-minute one-time edit eliminates a permanent tax.

### "The barrel is the feature's public API"

If yes — really, *yes* — keep it. The criterion above (it curates by *omitting* internals) is the test. If your barrel re-exports everything in the directory, it's not a public API; it's a shortcut.

### "Circular imports? My code doesn't have those"

Until you add a sibling that needs something from another sibling. The barrel makes the cycle invisible until something breaks at runtime in production. Direct imports surface cycles immediately at typecheck.

## Red Flags

Stop and re-read the rule when you see these:

- A directory containing `Foo.tsx` and `index.ts` where `index.ts` is one line: `export { default } from './Foo';`.
- More than 5 barrel files in a project with fewer than 50 source files. (Riboloven-club-priateli had 5 in 22 — 27% barrel rate.)
- An `import { ... }` whose path ends in a directory name when a same-named file exists inside.
- Turbopack HMR taking >2s on a one-line component edit.
- The phrase "let's add an `index.ts` for cleaner imports" in code review.
- A `find references` IDE result that points to the barrel before the source.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Cleaner imports" | "Cleaner" = "shorter for the typer." Costs everyone else. |
| "Easier to refactor — rename in one place" | The barrel just hides the rename. The IDE does the same job across files in ~1 second. |
| "Tree-shaking handles it" | Modern bundlers handle *some* barrels, not all. Turbopack specifically often can't. Run the analyzer. |
| "Convention in our codebase" | Conventions can be revisited. Adopt the no-barrel convention for new code; remove existing ones opportunistically. |
| "Public API for the feature" | If yes, keep that *one* barrel. Stop creating one per component directory. |
| "I need consistent import style" | Direct imports *are* a consistent style. |
| "Removing barrels means updating callers" | Once. Then never again. |

## Reference

Vercel published [a Turbopack tree-shaking note](https://vercel.com/docs/frameworks/nextjs/turbopack) that explicitly warns about barrel files defeating their import analysis. Next.js 13.4+ added `optimizePackageImports` in `next.config.js` as a workaround *for third-party barrels you can't change* — telling. The Next.js team's recommendation for *your own code*: don't write barrels in the first place.

The general anti-barrel argument is well-developed in the JS ecosystem; concrete reads:
- [Marvin Hagemeister, "Speeding up the JavaScript ecosystem — module resolution"](https://marvinh.dev/blog/speeding-up-javascript-ecosystem-module-resolution/)
- The TypeScript team's `isolatedModules: true` defaults (now mainstream) which subtly disincentivize barrels because they make namespace re-exports harder.

Adjacent rules: [[no-default-exports]] (the paired rule — named-only exports plus no barrels = every symbol has one canonical import path), [[dependency-inversion]] (the *real* feature boundaries — usually larger than per-component — are where ports live; per-component barrels aren't ports), [[single-responsibility]] (a barrel re-exporting unrelated symbols is itself a SRP violation), [[deep-modules]] (deep modules earn an `index.ts` as their narrow public face; shallow per-component barrels don't).
