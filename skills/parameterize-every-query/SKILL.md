---
name: parameterize-every-query
description: Use when building a SQL string with a template literal or concatenation that contains a variable (`WHERE email = '${email}'`). Use when passing a built-up command string to `exec`, `eval`, or a shell. Use when constructing a MongoDB filter, LDAP query, or any interpreter input from request data. Use when a value from the request reaches an ORM's raw-query escape hatch. Use when tempted to interpolate a column or table name from user input.
---

# Parameterize every query

## Overview

**Untrusted data never crosses into an interpreter as code. It is passed as a bound parameter, or through a safe API — never concatenated into the string the interpreter parses.** SQL, shell, NoSQL filters, LDAP: all are languages, and every one of them will execute data as instructions the moment you build the query by string-joining input.

One mental model covers the whole family: keep the *code* (the query template) separate from the *data* (the values), and let the driver combine them. The driver knows how to keep data as data; string concatenation does not.

## The iron rule

```
NEVER build an interpreter's input by concatenating or interpolating untrusted data.
Bind values as parameters, or use an API that separates code from data.
```

**No exceptions:**
- Not for "it's just an internal admin query"
- Not for "the value is a number, it can't be injected"
- Not for "I already validated the input"
- Not for "it's read-only, a `SELECT` is harmless"

## Why

The classic is SQL injection. Interpolating input into the query text lets the input *become* query text:

```ts
// ❌ The input is parsed as SQL. email = "' OR '1'='1" returns every row;
//    email = "'; DROP TABLE users; --" does what it says.
const rows = await db.execute(
  `SELECT * FROM users WHERE email = '${email}'`,
);
```

Parameterized, the value can never change the query's structure — the driver sends the template and the data separately, and the database treats the parameter as a pure value regardless of its contents:

```ts
// ✅ The template is fixed; `email` is bound. No value of `email` can alter
//    the statement — it is data, never code.
const rows = await db.execute(
  sql`SELECT * FROM users WHERE email = ${email}`, // driver binds ${email} as a parameter
);
// or, driver-level:
await client.query('SELECT * FROM users WHERE email = $1', [email]);
```

The same failure recurs in every interpreter. `child_process.exec(userInput)` hands a string to a shell that honors `;`, `|`, `$()`, and backticks. A MongoDB filter built from a raw request object lets `{ "$gt": "" }` slip in where a string was expected (operator injection). LDAP, XPath, and template engines all share the shape: **data concatenated into a language becomes that language.**

Validation (`parse-dont-validate`) reduces the blast radius by rejecting malformed input, but it is not the fix. A perfectly valid email string can still contain `'`; a valid-looking name can be `Robert'); DROP TABLE`. Shape-checking cleans the *input*; parameterization protects the *sink*. You need both, and the sink is non-negotiable.

## Detection

You are violating the rule if any of these are true:

- A SQL string is built with a template literal or `+` that contains a request-derived variable.
- A value reaches an ORM's raw escape hatch (`sql.raw`, `.$queryRawUnsafe`, `knex.raw` with interpolation) carrying interpolated input.
- `exec`, `execSync`, `eval`, `new Function`, or a shell command is constructed from input rather than passed argument-by-argument.
- A NoSQL filter is built by spreading or assigning a raw request object into the query (`find({ ...req.query })`).
- A column name, table name, sort direction, or `LIMIT` is interpolated from input (parameters bind *values*, not identifiers — see the identifier rule below).

## The pattern

### SQL: bind values; use `sql` template tags, not raw interpolation

```ts
// ❌ Raw interpolation — even "for a number", the value is concatenated as text.
await db.execute(sql.raw(`SELECT * FROM orders WHERE total > ${minTotal}`));

// ✅ Tagged template — the query builder parameterizes every ${...} interpolation.
await db.execute(sql`SELECT * FROM orders WHERE total > ${minTotal}`);
```

Prefer the ORM's query builder or a `sql` tagged template (which parameterizes automatically) over any `raw`/`Unsafe` API. When you must drop to raw SQL, pass values through the placeholder array (`$1`, `?`), never the string.

### Identifiers can't be bound — allowlist them

Parameters bind *values*, not column or table names. A sort column or direction from the request must be checked against a fixed set, never interpolated:

```ts
// ❌ Column and direction interpolated — "id; DROP TABLE users --" as a sort key.
const q = sql.raw(`SELECT * FROM users ORDER BY ${sortBy} ${dir}`);

// ✅ Map request input to known-safe identifiers; anything else is rejected.
//    Typed as a Record so indexing honestly yields `| undefined` under
//    noUncheckedIndexedAccess — no `as` cast asserting what wasn't checked.
const SORT_COLUMNS: Record<string, AnyColumn> = {
  name: users.name,
  createdAt: users.createdAt,
};
const column = SORT_COLUMNS[sortBy]; // AnyColumn | undefined
if (!column) throw new Error('invalid sort field'); // map to 400 at the boundary
const direction = dir === 'desc' ? desc : asc;
await db.select().from(users).orderBy(direction(column));
```

This is not a walk-back of the Iron Rule — the untrusted string never enters the query. A trusted constant, *selected by* the input, does. The allowlist exists because an identifier genuinely is query structure and therefore can't be bound.

### Shell: pass arguments as an array, never a command string

```ts
// ❌ A shell parses the whole string — filename = "x.txt; rm -rf ~" runs both.
import { exec } from 'node:child_process';
exec(`convert ${filename} out.png`);

// ✅ execFile with an args array — no shell, no metacharacter parsing.
import { execFile } from 'node:child_process';
execFile('convert', [filename, 'out.png']); // filename is a single argument, never code
```

Prefer `execFile`/`spawn` with an argument array over `exec`/a shell. If a shell is truly required, the input must never be interpolated into the command string — it reaches the shell only as a separately-passed, quoted argument.

### NoSQL: keep operators out of user-controlled positions

```ts
// ❌ Spreading the raw body lets { email: { "$ne": null } } match everything.
const user = await users.findOne({ ...req.body });

// ✅ Read named scalar fields and coerce type — an object where a string was
//    expected is rejected by the parse, so no operator can smuggle in.
const { email } = credentialsSchema.parse(req.body); // email: string
const user = await users.findOne({ email });
```

Here `parse-dont-validate` and this skill compose directly: parsing to a `string` type at the boundary is what stops an operator object from occupying a value position.

## Pressure resistance

### "I already validated the input, so it's safe"

Validation checks shape; it does not make concatenation safe. A valid string can contain a quote, a semicolon, a `$()`. The sink must parameterize regardless of how clean the input looked. Shape and sink are two defenses, not one.

### "It's a number, numbers can't be injected"

Only if it's actually a number at the sink. Interpolated into a string, a "number" is text you assumed was numeric — and an attacker controls whether it stays numeric. Bind it; then its type is enforced by the driver.

### "It's an internal/admin tool, not user-facing"

Internal tools take input too — from admins, from imports, from other services. Injection doesn't care about the caller's role, and admin tools often run with the highest privileges. The blast radius is *larger*, not smaller.

### "The ORM protects me automatically"

The ORM protects you *until you reach for the raw escape hatch* — `raw`, `Unsafe`, `$queryRawUnsafe`. Those exist precisely to bypass the protection; the moment you interpolate into one, you own the injection. Stay on the parameterized path.

### "I need the column name to be dynamic"

Then allowlist it against a fixed map (above). Identifiers can't be bound as parameters, so a closed set of known-safe columns is the only safe way to make them dynamic.

## Red flags

- A template literal or `+` inside a query string containing a variable.
- Any `raw`, `Unsafe`, or `$queryRawUnsafe` call with an interpolated value.
- `exec` / `execSync` / `eval` / `new Function` built from input.
- `find`/`findOne`/`update` receiving a spread of `req.query` or `req.body`.
- A column, table, or sort direction interpolated from request data.
- The reasoning "I validated it, so concatenation is fine."

**All of these mean: separate code from data — bind the value, allowlist the identifier, or pass args as an array.**

## Common rationalizations

| Excuse | Reality |
|---|---|
| "I sanitized the input" | Escaping by hand is error-prone and driver-specific. Bind parameters and let the driver do it. |
| "It's read-only" | A `SELECT` still exfiltrates data — `OR 1=1`, `UNION SELECT`. Read-only injection is a breach. |
| "The value can't contain quotes" | You can't prove that about untrusted input. Parameterize and stop reasoning about characters. |
| "Raw SQL is the only way to express this" | Raw SQL still takes a placeholder array. Raw ≠ interpolated. |
| "It's a trusted internal service calling us" | Services get compromised and send unexpected payloads. The sink defends regardless of source. |
| "Performance — building the string is faster" | Prepared statements are cached and typically *faster*, not slower. |

## Related

- `parse-dont-validate` — parsing cleans the input's shape at the boundary; this skill governs the sink. They compose: a value parsed to `string` can't smuggle a NoSQL operator into a value position.
- `authorize-every-access` — the other half of the input-defense pair; injection is *what* they send, authorization is *whether they may*.
- `secrets-handling` — a sibling security discipline; both keep untrusted or sensitive data out of places that will misinterpret it.

## Reference

- [OWASP Top 10:2025 — A05 Injection](https://owasp.org/Top10/2025/A05_2025-Injection/) — the injection family, renumbered from A03:2021.
- [OWASP Query Parameterization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html) — parameterized queries as the primary defense, with per-language examples.
- [OWASP Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Injection_Prevention_Cheat_Sheet.html) — the general "keep data out of the interpreter" model across SQL, OS command, LDAP, and NoSQL.
