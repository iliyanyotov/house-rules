---
name: revert-and-redo-on-confusion
description: Use when you've been refactoring or debugging the same problem for more than ~30 minutes without forward progress. Use when "let me just fix one more thing" has fired three times in a row. Use when the diff is getting larger and the test failures are getting weirder. Use when you stopped being sure what your code is supposed to do. Use when a teammate or AI agent asks "what state is the code in?" and the honest answer is "I don't know."
---

# Revert and Redo on Confusion

## Overview

**When you've been stuck on a change for ~30 minutes with no clear forward path, revert your working copy to the last known-good state and start over with the lessons you learned.** Don't dig deeper. Don't make "just one more attempt." The clean restart is almost always faster than the rescue dive.

This rule fights one of the most expensive failure modes in software: the spiral of partial changes that each make sense in isolation but compound into a working copy nobody — including you — understands.

This is the library's one **meta-process rule**. The others govern the shape of code; this one governs the shape of *the change-making process itself*. It's narrower in trigger and broader in applicability — every engineer hits the muddled-diff spiral; this rule names it and provides the exit.

## The Iron Rule

```
NEVER push past ~30 minutes of muddled diff without a hypothesis. Revert and redo from green.
```

**No exceptions:**
- Not for "I've already spent 30 minutes — reverting is wasted work"
- Not for "I'm so close — five more minutes"
- Not for "reverting loses my insight"
- Not for "senior devs push through"

## Why

After 30 minutes of frustrated work, three things are simultaneously true:

1. **The diff is muddled.** You added a parameter, then removed it; renamed a thing, then reverted; added a test, then commented it out. The current state is the *intersection* of half-baked attempts.
2. **Your mental model is wrong.** You started with a hypothesis. Either the hypothesis was wrong or the implementation didn't follow it. Continuing reinforces the wrong model.
3. **The cost has anchored you.** "I've spent 30 minutes; if I revert, I waste that time." This is the sunk-cost fallacy. The next 30 minutes you spend on the muddled copy will be *more* expensive than the next 30 minutes on a fresh start, because you'll be debugging *both* the bug and your own changes.

The clean restart is fast precisely *because* of what the last 30 minutes taught you:

- You now know which approach doesn't work.
- You know the shape of the data, the API of the dependency, the layout of the file.
- You know which test cases matter.
- Your hands have rehearsed the keystrokes; rewriting from scratch is 5× faster than the first time.

The 30 minutes weren't wasted — they were *paid learning*. The revert collects the dividend.

## Detection

You are violating the rule if any of these are true:

- You've been editing the same file for 30+ minutes and the diff is larger than when you started.
- You've added and removed the same lines/parameters/imports multiple times.
- Tests that used to pass are now failing in ways that surprise you.
- You're commenting out code "to check something" and not uncommenting it.
- You can't articulate *what* your current change is supposed to do anymore.
- You're patching symptoms instead of root cause and don't remember which is which.
- The phrase "almost there, just one more thing" has fired ≥3 times in this session.
- You started fixing one bug and are now in code unrelated to it.
- You're afraid to look at `git diff` because the diff is "complicated."

## The Pattern

### Commit the green state before you start changing

Every refactor or debugging session starts from a known-good state. **Commit it.** This is your safety harness:

```bash
# Before starting work — confirm everything passes.
npm test && npm run type-check && npm run lint

# Now commit.
git add -A
git commit -m "wip: green state before X refactor"
```

Now any disaster is one `git reset --hard HEAD` away from recoverable.

If you can't commit dirty state (work-in-progress on top of someone else's branch), `git stash` works too:

```bash
git stash push -m "wip: green state before X refactor"
```

### The 30-minute timer

Set a literal timer when you start a change you're unsure about. When it rings:

1. Run the tests.
2. If they're green and you're making forward progress — keep going, reset the timer.
3. If they're red or you're stuck — **stop**. Run `git diff` and read it. If the diff makes sense, keep going. If the diff *doesn't* make sense to you, revert.

The timer is the anti-anchor. Without it, you'll always feel "5 more minutes" away from a fix. With it, you have an external prompt to check in.

### The clean revert

```bash
# Nuclear option: discard everything since the safety commit.
git reset --hard HEAD

# Or, more surgical — revert just one file.
git checkout HEAD -- src/path/to/file.ts

# If you used stash:
git stash drop  # discard the muddled state
# (your safety state is the working tree, not the stash)
```

After the revert: **don't immediately start typing**. Sit for 60 seconds. Write down (in a scratch file, a comment, or your head) what you learned from the failed attempt:

- What approach doesn't work?
- Why didn't it work? (Wrong model? Missing knowledge? Hidden dependency?)
- What do you know now that you didn't know 30 minutes ago?

*Then* start again, applying the lesson.

### Smaller steps the second time

The second attempt is almost always *smaller* than the first. If your first attempt added 80 lines, the second attempt adds 12. The difference: you skip the dead ends.

This is a sign your mental model is right *this time*. If the second attempt also bloats, revert again — the model is still wrong.

### The "shadow workspace" trick

For exploratory work where you genuinely don't know if your approach will pan out:

1. Branch off: `git checkout -b experiment-A`
2. Do the work. Don't worry about commit hygiene; this is a scratch space.
3. If it works: cherry-pick the *good* changes onto main; abandon the branch.
4. If it doesn't: abandon the branch. **You haven't muddled `main` at all.**

The cost of branching is zero (in Git). The cost of *not* branching when you're unsure is the muddled diff you'll have to revert.

### A worked example

Suppose you're refactoring a stateful component from useState sprawl to a discriminated union. You start:

```
00:00 — Commit green state.
00:00 — Begin refactor. Replace 3 useState with 1.
00:10 — Tests fail; a timer cleanup is misbehaving.
00:15 — Try one fix. Tests still failing differently now.
00:25 — Realize the onExpire callback was reading stale state. Try a ref.
00:32 — Tests passing, but a downstream consumer is broken.
00:45 — Fixing the consumer. Add a prop. Then remove it. Then add a different one.
01:00 — TIMER. Run `git diff`. The diff is 200 lines of half-finished changes.
       Some are unrelated (you renamed a file you didn't mean to).
```

At 01:00 you have two options:

- **Bad:** keep going. Your `git diff` doesn't compose any coherent story. You'll spend the next hour untangling.
- **Good:** `git reset --hard HEAD`. Note: "I learned the timer's onExpire reads stale state because the useEffect captured the closure. The fix is to use a ref for the latest value. The downstream consumer needs its own concern split — that's a separate PR."

Then redo. The second pass takes 25 minutes because you skip:

- The wrong fix at 00:15 (which fought stale-closure with a useEffect change).
- The accidental file rename.
- The downstream half-fix.

**Net time:** 60 min muddled + 60 min untangle = 120 min. Or: 60 min muddled + 25 min clean redo = 85 min. The revert paid back instantly.

## Pressure Resistance

### "I've already spent 30 minutes — reverting is wasted work"

The work isn't wasted; it's the cost of learning. You now know the approach that doesn't work. The next 30 minutes spent untangling the muddled state will cost *more* than the next 30 minutes redoing cleanly, because you'll be debugging your own confusion on top of the original problem.

### "I'm so close — five more minutes"

You're not close. The "five more minutes" feeling has fired three times already. It's a signal of frustration, not progress. The hands feel close because they remember the work; the mind feels close because it has invested. Neither is evidence of being close.

### "Reverting loses my insight"

Write it down before reverting. A 30-second sentence ("the issue was X; the fix needs Y") is more valuable than the 200 lines of failed-attempt code. Your insight isn't in the diff; it's in your head.

### "I don't have a clean commit to revert to"

That's the bug. Commit before you start work, not after. If you didn't this time, learn the rule and adopt it next time. For *now*, do the best you can: stash the muddled state (`git stash`) so it's recoverable, then `git stash apply` later if you want to mine it for ideas.

### "Reverting is for inexperienced devs; senior devs push through"

The opposite is true. Senior devs revert *more often* than juniors — they're calibrated to recognize when they're stuck, and they know the time cost of the muddled-state spiral. Juniors push through; seniors restart.

### "I'll lose my flow state"

Flow built on a wrong model produces wrong work. The "flow" of the last 30 minutes was the *appearance* of progress, not progress. Real flow comes from the second-attempt clarity, when each step has obvious next steps.

## Red Flags

- You've changed your mind about the approach mid-edit.
- You're reading the test output and the result doesn't match either your code OR the original code.
- You can't summarize what your in-progress change does in one sentence.
- You've reverted the same line of code 3+ times.
- The phrase "let me check one more thing" has fired without making progress.
- You're afraid of `git diff`.
- You've added `// FIXME` or `// TODO` markers that you don't remember writing.
- A colleague or AI assistant asks "what are you trying to do?" and your answer is convoluted.
- 30 minutes has passed since the last successful test run.

**All of these mean: stop, revert, capture the lesson in words, then start fresh.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "I'm 95% done" | If you were 95% done, you wouldn't be feeling stuck. The 95% feeling is the *first* place the spiral lies. |
| "Reverting wastes my work" | The *learning* isn't in the diff. Capture it in words and revert the code. |
| "It's faster to fix than to redo" | When you're confused, the redo is faster than the fix. The redo skips the dead ends. |
| "I haven't committed in a while" | Then committing more often is the lesson. Commit hygiene is the prerequisite to using this rule. |
| "I'll lose ideas I had during the attempt" | Write them down. The ideas are cheap; the muddled code is expensive. |
| "Senior engineers don't revert" | Senior engineers revert *more* than juniors. They're calibrated to the spiral. |

## Reference

- Kent Beck, *Test-Driven Development by Example* (2002) — when the test goes from red to *more* red (worse failures), the move is to revert the last change. Beck's framing: *"When in doubt, throw it away. It only took 10 minutes to write."*
- Michael Feathers, *Working Effectively with Legacy Code* (2004) — implicit in the "scratch refactor" pattern: try a refactor, learn from how it fails, throw the work away, plan the real refactor with the learning.
- Andy Hunt & Dave Thomas, *The Pragmatic Programmer* (1999) — *"Tracer bullets"*: try a thin slice end-to-end, learn from where it fails, then aim properly. The revert is the act of *not aiming a second time without re-zeroing*.
