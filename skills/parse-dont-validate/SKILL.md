---
name: parse-dont-validate
description: Use when accepting external input — request bodies, form submits, env vars, API responses, message payloads. Use when tempted to call a schema parser more than once in a request path, or to re-check a field three layers down. Use when reaching for `as` to coerce `unknown` into a domain type. Use when writing a guard function that returns `boolean` and then using the input as the narrow type.
---

# Parse, Don't Validate

## Overview

**At every system boundary, transform `unknown` into a narrowed, branded type — once. Pass the narrowed type onward. Never re-validate downstream.**

The boundary parses. The interior trusts. A function whose parameter type already encodes the proof you need does not need to re-prove it. If it feels like it does, the parameter type is wrong.

## The Iron Rule

```
NEVER re-validate a value the type system already considers proven.
```

**No exceptions:**
- Not for "defensive programming"
- Not for "belt and suspenders"
- Not for "just to be safe"
- Not for "what if someone breaks the boundary later"

## Why

Validation throws away information. After `if (!isValidEmail(input)) throw` you still hold a `string`, indistinguishable at the type level from any other string. Three layers down, someone re-checks. Or doesn't.

Parsing preserves information. After `const email = Email.parse(input)` you hold an `Email`, a type the compiler treats as distinct from `string`. Three layers down, the type signature says `(email: Email) => ...` and the compiler enforces it.

A second `.parse()` is permission for the rest of the codebase to write code that doesn't trust its inputs. Once that pattern exists, every function in the call chain might or might not be a boundary. Reviewers can't tell. New code calcifies the ambiguity.

## Detection

You are violating the rule if any of these are true in a single request path:

- The same schema is `.parse()`ed in two places.
- A function body starts with `if (typeof x !== 'string') throw` for an `x` typed as `unknown` that came from somewhere typed.
- A guard returning `boolean` is followed by use of the input as the narrow type.
- An `as User` silences a complaint after a manual check.
- A schema is imported into a domain file (not a boundary file).
- A `try { Schema.parse(x) } catch {}` falls through to default values — that's hoping, not parsing.

## The Pattern

### Webhooks and external responses — `safeParse` at the edge

```ts
const PaymentEvent = z.object({
  id: z.string(),
  type: z.string(),
  amountCents: z.number().int().positive(),
});

export async function handlePaymentWebhook(rawBody: unknown) {
  const result = PaymentEvent.safeParse(rawBody);
  if (!result.success) {
    return { status: 400, body: { error: 'invalid_payload' } };
  }

  // From here on, `result.data` is typed and trusted.
  await routePaymentEvent(result.data);
  return { status: 200, body: { ok: true } };
}
```

Untrusted external input gets `safeParse` + a structured response. The interior — `routePaymentEvent` — accepts the parsed type and never re-checks.

### Env vars are a boundary

```ts
const Env = z.object({
  DATABASE_URL: z.string().url(),
  STRIPE_SECRET_KEY: z.string().startsWith('sk_'),
  REDIS_URL: z.string().url(),
});

export const env = Env.parse(process.env);
```

One parse, at module load. Every consumer reads `env.DATABASE_URL` (typed as `string`, narrowed by the schema) — no `process.env.X ?? 'localhost'` fallbacks scattered through the code.

## Pressure Resistance

### "It's only one extra `.parse()`, what's the harm?"

The harm isn't CPU — it's that the second parse is permission to write code that doesn't trust its inputs. One per boundary. If you need to parse again, you've found a *new* boundary; name it.

### "I'll just `as` it, the structure obviously matches"

`as` is an assertion to the compiler that you've done the work. If you haven't done the work, you're lying to the compiler. The bug surfaces in production, not in review. If the source is `unknown`, the answer is `Schema.parse(...)`. If the source is already typed, you don't need `as` either — pass it through.

### "The function already takes `User`, so it must be valid"

Then trust the parameter type. Stop re-checking. If the parameter type is actually `unknown` or `Partial<User>`, change it to `User`. The fix is in the signature, not the body.

### "But the boundary parses *some* fields, and I need to derive more"

Derivation isn't re-validation. `const total = items.reduce(...)` is fine. `Schema.parse()` on a value the type system already knows is `Item[]` is not.

### "Defensive programming says check at every layer"

That's about boundaries you don't control. Within one process, the boundary is the boundary. Defense in depth applies between systems, not between functions.

## Red Flags

- A function signature with `unknown` or `any` more than one call away from a route handler / action / webhook / env load.
- A schema `.parse()` inside a service, a repository, or a database adapter.
- A guard function returning `boolean` followed by use of the input as a narrowed type.
- A schema imported into a domain file rather than a boundary file.
- The phrase "just to be safe" or "defense in depth" applied within a single process.
- A `try { Schema.parse(x) } catch {}` that falls through to defaults.

**All of these mean: the boundary is the wrong place — find it and parse there, once.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "Defensive programming says check at every layer" | Defense in depth is for boundaries you don't control. Within one process, the boundary is the boundary. |
| "What if someone refactors and breaks the boundary later?" | Tests catch that. Re-validating everywhere just hides the regression. |
| "Zod parsing is fast, it doesn't matter" | The cost isn't speed. The cost is unclear ownership of who-trusts-what. |
| "I want belt and suspenders" | Belt: parse at the boundary. Suspenders: brand the type. You already have both. |
| "It's hard to thread the type through every call" | If it's hard, the boundary is wrong — not "you need more checks." |
| "Logs need the raw input for debugging" | Log the raw input *at the boundary*, before parsing. Then parse. |

## Reference

- Alexis King, ["Parse, don't validate"](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) (2019) — the original framing. Haskell, but the TypeScript application is a direct translation: parsing returns a *new type* that carries the proof; validation returns a *boolean* and throws the proof away.
- Yaron Minsky, *Effective ML* — "Make illegal states unrepresentable." The neighboring rule that parsing makes possible.
