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
- A DB/ORM row exposes a field as `Json` / `unknown` / `any` (e.g. a Prisma `Json?` column), so every reader must `.parse()` it. The DB read *is* the boundary — parse the column once in the repository and return the narrowed type, so callers get the proven type, not the raw row.

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

### One parse at the controller; the layers below trust the type

This is the load-bearing pattern in a layered backend. The schema's **output** type *is* the domain type, branded IDs are minted in the same transform, and the controller is the only place that touches `unknown`. Service and repository receive `User` and `OrderId` — already proven — so they have nothing left to validate.

```ts
type UserId = string & { readonly __brand: 'UserId' };

const userSchema = z
  .object({
    id: z.string().uuid(),
    email: z.string().email(),
    displayName: z.string().min(1).max(120),
  })
  .transform((row) => ({
    // The one legitimate `as`: branding a value we just proved is a UUID.
    id: row.id as UserId,
    email: row.email,
    displayName: row.displayName,
  }));

// The PARSED type — z.output, not z.input. This is the domain type.
type User = z.output<typeof userSchema>;
```

The boundary is the controller. It is the only layer that sees `unknown`:

```ts
// controller.ts — the ONLY parse boundary in this path
export async function createUserController(req: Request): Promise<Response> {
  const result = userSchema.safeParse(await req.json());
  if (!result.success) {
    return Response.json({ error: 'invalid_user' }, { status: 400 });
  }

  // result.data is `User` — branded, narrowed, proven. Hand it inward.
  const user = await createUser(result.data);
  return Response.json(user, { status: 201 });
}
```

Every layer below receives the domain type and re-proves nothing:

```ts
// service.ts — takes `User`, not `unknown`. No schema import. No re-parse.
export async function createUser(user: User): Promise<User> {
  await sendWelcomeEmail(user.email); // `email` already valid; no re-check
  return insertUser(user);
}

// repository.ts — also takes `User`. The id is already a `UserId`.
export async function insertUser(user: User): Promise<User> {
  await db.insert(users).values(user);
  return findUserById(user.id); // `user.id` is `UserId`; the query helper demands it
}
```

Contrast the anti-pattern — the same field re-checked at every depth, each layer distrusting the last:

```ts
// ❌ The boundary failed to carry the guarantee, so everyone re-proves it.
export async function createUserController(req: Request) {
  const body = (await req.json()) as { email?: string }; // lie #1
  return createUser(body);
}
export async function createUser(body: { email?: string }) {
  if (!body.email?.includes('@')) throw new Error('bad email'); // re-check #1
  return insertUser(body);
}
export async function insertUser(body: { email?: string }) {
  if (!body.email) throw new Error('missing email'); // re-check #2 — three layers deep
  // ...
}
```

One parse at the top; the type system carries the proof inward. The fix for a downstream `if (!user.email)` is never another check — it's making the boundary mint a type that can't be missing one.

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

A lazy typed accessor — `getEnv(key): T => process.env[key] as T` — is *not* this pattern, even though it looks typed. The `as` asserts a type that was never checked: `STRIPE_SECRET_KEY` typed `string` is never verified to start with `sk_`, and a `number`-typed var is still the raw string at runtime. That's the `as`-coercion lie, just centralized behind a helper. Parse once with a schema so the narrowed type is *earned*, not asserted.

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
- The same schema `.parse()`ed at many independent sites because the source type (often a Prisma `Json` column) never carried the narrowed type outward — parse it *once* at the repository boundary so the type flows, instead of every reader re-parsing.
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

## Related

- `branded-ids` — branding is the boundary's typed output
- `make-illegal-states-unrepresentable` — parsing yields the narrowed/union type
- `no-any-escape-via-unknown-or-never` — narrow from `unknown`, never `any`
- `prefer-type-inference-annotate-at-boundaries` — annotate the boundary, infer inward

## Reference

- Alexis King, ["Parse, don't validate"](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) (2019) — the original framing. Haskell, but the TypeScript application is a direct translation: parsing returns a *new type* that carries the proof; validation returns a *boolean* and throws the proof away.
- Yaron Minsky, *Effective ML* — "Make illegal states unrepresentable." The neighboring rule that parsing makes possible.
