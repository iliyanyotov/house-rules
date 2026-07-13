---
name: pin-and-verify-dependencies
description: Use when adding a dependency with a floating range (`^`/`~`) or without committing the lockfile. Use when a CI job runs a bare `bun install` or `npm install` rather than a frozen install (`bun install --frozen-lockfile`, `npm ci`). Use when reviewing a PR that adds a package you haven't vetted, or bumps many transitive versions at once. Use when a dependency ships a post-install script, or `bun pm untrusted` shows blocked scripts nobody has reviewed. Use when the lockfile is absent from the repo or not enforced in the build.
---

# Pin and verify dependencies

## Overview

**Builds install the exact dependency tree the lockfile records — committed, enforced, and reproducible — and new packages are vetted before they enter it.** A dependency is code you did not write, running with your process's privileges. Left to float, it updates itself between builds, and one compromised patch release becomes your compromise.

The supply chain is now a primary attack vector, not a theoretical one: malicious versions published to real registries have self-propagated through hundreds of packages by riding auto-installed updates. The defense is boring and effective — reproducible installs and a review gate.

## The default rule

This is a **default with a clear boundary**, not an exceptionless invariant: pin and verify what you can control in a pull request, and leave the org-scale machinery (SBOM generation, signing infrastructure, registry policy) to the platform.

```
Prefer: commit the lockfile, install it exactly in CI, and review a
dependency before it enters the tree. Do not float versions silently.
```

The rule governs the surface a developer touches in a PR — `package.json`, the lockfile, the CI install step, and post-install scripts. It deliberately stops short of SBOM tooling, Sigstore/SLSA provenance, and organizational SCA policy: those are infrastructure, owned elsewhere, and folding them in here would turn a house rule into a program.

## Why

The mechanism of a modern supply-chain compromise is simple: a maintainer account is phished or a malicious version is published, and every project with a floating range (`^1.2.3`) pulls the bad version on its next install — no code change, no review, no signal. The September 2025 "Shai-Hulud" npm worm spread through 180+ packages this way; the pattern recurs regularly.

Three properties turn a floating dependency into an incident:

1. **Silent updates.** `^` and `~` resolve to "newest compatible" at install time. Two builds of the same commit can install different code. A malicious patch lands with no diff in your repo.
2. **Install-time execution.** A package's `postinstall` script runs arbitrary code on the developer's machine and in CI — before a single line of your app runs. This is where credential-stealing payloads execute.
3. **Transitive reach.** You vet your direct dependencies; the compromise is usually three levels down, in a package you've never heard of but that runs with the same privileges.

Reproducible installs neutralize (1): the lockfile pins every transitive version, and `npm ci` installs *exactly* that tree or fails. A review gate addresses (3): a new package — direct or a surprising transitive bump — gets a human look before it merges. Disabling or scrutinizing install scripts blunts (2).

## Detection

You are at risk if any of these are true:

- The lockfile (`bun.lock`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`) is not committed, or is in `.gitignore`.
- CI runs `bun install` / `npm install` / `yarn` / `pnpm install` rather than the frozen-install variant (`bun install --frozen-lockfile`, `npm ci`, `pnpm install --frozen-lockfile`, `yarn install --immutable`).
- A dependency was added or bumped without the lockfile change appearing in the same PR.
- A PR bumps a large number of transitive versions with no note on why.
- Install scripts run unconditionally in CI (no `--ignore-scripts` on untrusted installs).
- A new direct dependency merged without anyone stating they looked at what it is, who maintains it, and what it pulls in.

## The pattern

### Commit the lockfile; install it frozen in CI

The lockfile is the source of truth for *what actually runs*. It must be committed and enforced — a build that can't satisfy it exactly should fail, not silently resolve something new.

```jsonc
// ❌ CI resolves fresh each run — "newest compatible" can differ build to build.
// package.json script:
{ "scripts": { "ci": "npm install && npm test" } }
```

```jsonc
// ✅ A frozen install uses exactly the committed lockfile, or errors.
//    Deterministic: the same commit installs the same bytes every time.
{ "scripts": { "ci": "bun install --frozen-lockfile && bun test" } }
// npm equivalent: "npm ci && npm test"
```

`bun install --frozen-lockfile` (and its `npm ci` / pnpm `--frozen-lockfile` / yarn `--immutable` equivalents) refuses to proceed if `package.json` and the lockfile disagree — which is exactly the signal you want when something changed the tree unexpectedly.

### Neutralize install-time code execution

Post-install scripts execute before your app does. On untrusted or first-time installs, skip them; enable only for the specific packages that genuinely need a build step.

Bun ships this rule as its default: dependency lifecycle scripts do **not** run unless the package is allowlisted in `trustedDependencies` in your `package.json` (plus a small built-in allowlist of popular native packages). `bun pm untrusted` lists the packages whose scripts were blocked; `bun pm trust <pkg>` opts one in deliberately. That is exactly the allowlist posture the other managers need flags to reach:

```bash
# ✅ Bun: blocked by default — inspect, then trust specific packages.
bun pm untrusted            # what wanted to run and didn't
bun pm trust sharp          # deliberate, reviewable opt-in

# ✅ npm/pnpm: skip lifecycle scripts by default; opt specific packages back in.
npm ci --ignore-scripts     # CI (local: npm install --ignore-scripts)
# pnpm: onlyBuiltDependencies allowlists which packages may run scripts

# ❌ npm install — postinstall from any dependency (or transitive dep) runs immediately.
```

Most dependencies don't need a post-install script at all; the ones that do (native builds) are a short, reviewable list. Note that npm's `--ignore-scripts` also skips your *own* project's lifecycle scripts (`prepare`, `postinstall`) — run those explicitly as a separate build step where needed (Bun's default only gates *dependency* scripts, so this caveat doesn't apply there).

### Vet a dependency before it enters the tree

Adding a package is a code-review event, not a formality. Before it merges, someone states: what it is, who maintains it, how much transitive weight it drags in, and whether a smaller/stdlib option exists (this is where `yagni` and `kiss` pull their weight — the safest dependency is the one you didn't add).

```
Reviewer checklist for a new direct dependency:
  - Is it actively maintained, with a plausible download/issue history?
  - How many transitive packages does it add? (check the lockfile diff)
  - Does it ship an install script? Why?
  - Is there a standard-library or already-present alternative?
```

### Treat `audit` as a signal, not a gate you can trust

`npm audit` reports *known* advisories. It is worth running and worth acting on — but a clean audit is not proof of safety (a brand-new malicious version has no advisory yet), and a noisy audit is not always an emergency. Use it to surface known-bad, not as the whole defense.

## Pressure resistance

### "Floating ranges get us security patches automatically"

They also get you *malicious* patches automatically — the same mechanism, no discrimination. Patches you *chose* (a reviewed lockfile bump) give you the fix without the open door. Automate the *proposal* (a bot PR), keep the *merge* human.

### "The lockfile causes merge conflicts, it's easier without it"

Lockfile conflicts are a two-minute regeneration; a non-reproducible build is a debugging session and a security hole. The conflict is the cost of knowing exactly what runs — pay it.

### "`npm install` and `npm ci` are basically the same"

They are not. `install` may *update* the lockfile to satisfy `package.json`; `ci` installs the lockfile *exactly* or fails. In CI you want the version that fails loudly when the tree drifts, not the one that quietly resolves something new.

### "We can't review every transitive dependency"

You can't, and the rule doesn't ask you to. It asks you to pin them (so they don't change under you) and to review *direct* additions and *surprising* transitive bumps. Pinning is what makes the un-reviewable tree at least *stable*.

### "This is CI/ops config, not coding"

Part of it is — and that part is explicitly scoped in. `package.json` ranges, the committed lockfile, and the install step are all in the repo, changed in PRs, and reviewed like code. The rule stops at the org-infrastructure line on purpose.

## Red flags

- No lockfile committed, or lockfile in `.gitignore`.
- `npm install` (not `npm ci`) in a CI or Dockerfile build step.
- A dependency bump PR with no lockfile change, or a lockfile change with no `package.json` change and no explanation.
- A new package merged with zero discussion of what it is.
- Install scripts running on untrusted installs.
- `^`/`~` ranges treated as "set and forget."

**All of these mean: commit and enforce the lockfile, install it frozen, and put a human between a new dependency and the tree.**

## Common rationalizations

| Excuse | Reality |
|---|---|
| "Pinning means we miss security updates" | You don't miss them — you *choose* them via reviewed bumps. You also miss the malicious ones. |
| "The registry would catch a malicious package" | Registries catch some, late. Worms have shipped through hundreds of packages before takedown. |
| "It's a tiny, popular package, it's fine" | Popular packages are the *highest-value* targets, and "tiny" often means one maintainer, one phished account. |
| "audit is green, we're covered" | `audit` only knows *published* advisories. A fresh compromise is invisible to it. |
| "We'll pin it later if there's a problem" | "Later" is after the incident. Reproducibility is cheap up front, expensive in forensics. |
| "Disabling install scripts breaks things" | It breaks the few packages that need a build — allowlist those, skip the rest. |

## Related

- `yagni` — the safest dependency is the one you didn't add; question whether a package earns its transitive weight before pulling it in.
- `kiss` — a few lines of your own code can beat a dependency that drags in a subtree you now have to trust.
- `secrets-handling` — install-time scripts run with access to the environment; a compromised dependency is a credential-exfiltration path, which is why script execution is gated here.
- `small-changesets` — a dependency bump is its own reviewable concern; don't bury a version change inside an unrelated PR.

## Reference

- [OWASP Top 10:2025 — A03 Software Supply Chain Failures](https://owasp.org/Top10/2025/A03_2025-Software_Supply_Chain_Failures/) — renamed and broadened from "Vulnerable and Outdated Components" (A06:2021) and elevated to #3.
- [OWASP NPM Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/NPM_Security_Cheat_Sheet.html) — lockfile commitment, `npm ci`, and controlling install scripts.
- [OWASP Software Supply Chain Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Software_Supply_Chain_Security_Cheat_Sheet.html) — reproducible, verified installs and dependency vetting.
