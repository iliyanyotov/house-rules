---
name: secrets-handling
description: Use when accepting an API key, database URL, signing secret, webhook signature, OAuth token, or any other credential. Use when configuring `process.env.X` reads or a typed env loader. Use when adding a field to an error tracker `captureException` payload, a `console.log`, an error message, or an error response body. Use when committing a file that touches `.env`, `.env.example`, or anything in `src/config/`. Use when a credential might end up in a URL, a query parameter, a JWT body, or a log line.
---

# Secrets Handling

## Overview

**Secrets enter the process via environment variables, are read once at startup through a typed schema, and never leave that boundary** — not in logs, not in error messages, not in URLs, not in error response bodies, not in client bundles, not in `git`.

A leaked secret is not a bug you fix — it's a secret you rotate. The work to prevent the leak is always cheaper than the work to rotate after a breach.

## The Iron Rule

```
NEVER include a secret in any payload that crosses a boundary — log, response, URL, bundle, commit.
```

**No exceptions:**
- Not for "it's just the staging key"
- Not for "the error message is more helpful with the value"
- Not for "logs are internal-only"
- Not for "we'll redact it later"

## Why

Three categories of leak account for nearly all real-world incidents:

1. **Commit leaks.** `.env` checked in by mistake; secrets pasted into code "temporarily." Once public on a hosted repo, the secret is compromised even if reverted.
2. **Log leaks.** A `console.log(config)` or `captureException(err, { extra: { apiKey } })` ships the secret to a third party. Error trackers and log aggregators retain logs for weeks; one bad log line means rotation.
3. **Response leaks.** An error message like `Failed to call provider with key sk_live_...` is returned to a user, who shares the screenshot. Or a JWT carries a secret in its payload (JWTs are signed, not encrypted — the body is base64-decoded by anyone).

The pattern across all three: **secrets travel further than developers expect.** A `console.log` written for debugging ships to production. A breadcrumb captures form values. A 500 error includes the request body. Defaulting to *"never include credentials in any payload that crosses a boundary"* is the only durable defense.

## Detection

You are violating the rule if any of these are true:

- A secret literal (`"sk_..."`, `"AIza..."`, `"ghp_..."`, etc.) appears anywhere in the source tree, including comments, tests, commented-out code.
- A `process.env.X` is read outside a typed env loader.
- An error-tracker `captureException(..., { extra })` includes `req.body`, the entire request, or `config` objects that may carry credentials.
- A `console.log` outputs an object that contains an env value or anything derived from one.
- An HTTP error response body echoes the request body or includes a stack trace with bound variables.
- A `.env*` file is missing from `.gitignore`, or `.env.example` has *real* values rather than placeholders.
- A client-side bundle environment variable (anything client-readable) includes something that *isn't* meant to be public.
- A secret is sent as a URL query parameter rather than an `Authorization` header.

## The Pattern

### Parse all secrets at module load via a typed schema

```ts
import { z } from 'zod';

const Env = z.object({
  PAYMENT_SECRET_KEY: z.string().startsWith('sk_'),
  DATABASE_URL: z.string().url(),
  ERROR_TRACKER_TOKEN: z.string().min(1),
});

export const env = Env.parse(process.env);
```

The schema fails the build if a required secret is missing. Server-only vs client-exposed vars are kept structurally separate (a server secret in a client-bundle variable is a build error, not a runtime surprise).

### Constant-time comparison for signature verification

```ts
import { createHash, timingSafeEqual } from 'node:crypto';

// ❌ Vulnerable to timing attack — string equality short-circuits on first mismatch.
function secretsMatchInsecure(supplied: string): boolean {
  return supplied === env.CRON_SECRET;
}

// ✅ Constant-time compare via hash buffers of equal length.
function sha256(s: string): Buffer {
  return createHash('sha256').update(s).digest();
}

function secretsMatch(supplied: string | null | undefined): boolean {
  if (!supplied) return false;
  return timingSafeEqual(sha256(supplied), sha256(env.CRON_SECRET));
}
```

The `sha256` wrapping ensures equal-length buffers (a precondition of `timingSafeEqual`) and prevents leaking the secret's length via a length-mismatch error.

### Don't leak secrets in requests, errors, or logs

Two leaks to watch for in one call: the secret in the URL, and the secret in the error message. They usually appear together.

```ts
// ❌ Two leaks in one call:
//    1. Secret in the URL — captured by access logs, referrer headers,
//       error-tracker breadcrumbs, and (in client code) browser history.
//    2. Secret in the error message — propagates to logs, the error
//       tracker, and any response body that includes err.message.
async function callApi() {
  try {
    return await fetch(`https://api.example.com/data?api_key=${env.API_KEY}`);
  } catch (err) {
    const msg = err instanceof Error ? err.message : String(err);
    throw new Error(`Failed with key ${env.API_KEY}: ${msg}`);
  }
}

// ✅ Secret in the Authorization header (not the URL); error names the
//    operation, not the credential. The `cause` chain preserves the
//    underlying error for operators without copying secrets into the
//    user-visible message.
async function callApi() {
  try {
    return await fetch('https://api.example.com/data', {
      headers: { Authorization: `Bearer ${env.API_KEY}` },
    });
  } catch (err) {
    throw new Error('example.com call failed', { cause: err });
  }
}
```

When a legacy API *requires* the key in the URL, the request shape is forced — but the observability discipline still applies: don't include those URLs in breadcrumb extras, don't log them, and strip them before re-throwing.

Rule of thumb: error messages, log lines, and response bodies are written by you; the secret is something you possess. Never let possession leak into output.

### Error-tracker payloads — categorize what goes where

```ts
// ❌ Secrets in extras; request body likely contains tokens.
errorTracker.captureException(err, {
  extra: {
    apiKey: env.PAYMENT_SECRET_KEY,
    requestBody: await req.text(),
  },
});

// ✅ Secrets → nowhere. PII → contexts (with intent). Operational info → tags (low cardinality).
errorTracker.captureException(err, {
  tags: { route: 'webhook/payment', stage: 'verify' },
  contexts: {
    client: { ip: request.headers.get('x-forwarded-for') },
  },
});
```

Three buckets, three policies: secrets never, PII deliberately, operational info freely.

### `.env` discipline

```gitignore
.env
.env.local
.env*.local
.env.development
.env.production
```

`.env.example` is committed with placeholder values only:

```bash
PAYMENT_SECRET_KEY=sk_test_REPLACE_ME
DATABASE_URL=postgres://user:pass@localhost:5432/db
```

The `.env.example` documents *what* keys exist without exposing their values.

### Defense in depth — three layers, no single point of failure

The patterns above each guard one stage. A credential survives a mistake at any *one* stage only if the other layers also hold. Defend a secret at rest, in the type system, and at the egress boundary.

**Layer 1 — at rest: store a vault *reference*, not the secret.** The committed config holds a pointer the deploy resolves; the literal never touches the repo. `.env.example` lists names only.

```bash
# .env.example — names and a reference shape, never a value
PAYMENT_API_KEY=op://vault/payment-api/credential   # secrets-manager ref, resolved at deploy
```

```bash
# .env (gitignored) — also just the reference; the deploy injector resolves it
PAYMENT_API_KEY=op://vault/payment-api/credential
```

The process sees the resolved secret only at runtime. If the repo leaks, an `op://` ref is not a credential. (Any secrets-manager ref format works; `op://vault/item/field` is a widely-known shape.)

**Layer 2 — in the type system: brand the secret at the parse boundary** so it can't be passed where a plain `string` is expected. The brand is applied once, where `unknown` becomes typed (see `parse-dont-validate`), and flows from there.

```ts
import { z } from 'zod';

declare const brand: unique symbol;
type PaymentApiKey = string & { readonly [brand]: 'PaymentApiKey' };

const Env = z.object({
  // Brand at the boundary: downstream code receives PaymentApiKey, not string.
  PAYMENT_API_KEY: z.string().min(1).transform((s) => s as PaymentApiKey),
});

export const env = Env.parse(process.env);

function charge(amount: number, key: PaymentApiKey): Promise<void> { /* ... */ }

charge(1000, env.PAYMENT_API_KEY); // ✅ only the branded value type-checks here
charge(1000, 'sk_live_oops');      // ❌ compile error: string is not PaymentApiKey
charge(1000, user.displayName);    // ❌ compile error: a plain string can't slip into the key slot
```

**Layer 3 — at egress: a structured error that strips sensitive context out of client-facing 5xx responses.** Operators get the full error; clients get a context-free view, so a secret placed in `context` for debugging never rides out in a response body.

```ts
type ErrorContext = Record<string, unknown>;

class AppError extends Error {
  constructor(
    message: string,
    readonly status: number,
    private readonly context: ErrorContext = {},
    options?: { cause?: unknown },
  ) {
    super(message, options);
  }

  // Full detail — logs / error tracker only, never a response body.
  asJson() {
    return { message: this.message, status: this.status, context: this.context };
  }

  // Context-stripped — safe to return to clients on a 5xx.
  asJsonWithoutContext() {
    return { message: this.message, status: this.status };
  }
}

// At the boundary: log the full error internally, return the stripped view outward.
function toResponse(err: AppError): Response {
  logger.error(err.asJson()); // context (may hold a key) stays internal
  const body = err.status >= 500 ? err.asJsonWithoutContext() : err.asJson();
  return Response.json(body, { status: err.status });
}
```

No layer is sufficient alone: a branded type doesn't stop a committed secret; a vault ref doesn't stop `context` leaking in a 5xx; a context-stripped error doesn't stop the wrong `string` reaching the key slot. Together they make a single slip survivable.

## Pressure Resistance

### "It's a dev/staging key, it doesn't matter"

It matters. Dev keys have rate limits attackers can exhaust to DOS your dev environment, often share infrastructure with prod (test-mode keys can still hit live metering), and are practice for the same discipline applied to prod keys. Adopt the discipline universally.

### "The error message is more helpful with the value"

The user doesn't need the value to understand the error. Operators have logs (which should be redacted) and the error tracker for internal context. Errors visible to users say *what failed*, never *with what credential*.

### "Logs are internal-only"

They aren't. Error trackers, log aggregators, cloud log services — all third parties or shared internal systems. Logs are forwarded to syslog, ELK, BigQuery, S3. Treat the boundary as "anywhere `console.log` reaches" — which is many places.

### "We use encryption-at-rest, secrets are protected"

Encryption-at-rest doesn't help once a secret is in a process. The rule is about runtime handling: once parsed into `env`, the secret stays there. Encryption tools protect storage; this rule protects use.

### "Rotating is a hassle"

It's a smaller hassle than the breach. Secret managers make rotation a 5-minute operation. Practice rotation quarterly so it's routine, not panicked.

## Red Flags

- Any literal that looks like a secret in source (`sk_live_*`, `AIza*`, `ghp_*`, `xoxb-*`, `eyJhbGc*`).
- `process.env.X` outside a typed env file.
- An error-tracker `extra` field with `request`, `body`, `headers`, or `config` keys.
- A `console.log(env)` or `console.log(config)`.
- A URL with `?key=`, `?token=`, `?api_key=`, `?password=`.
- A `.env.example` with values that look real (not placeholder-shaped).
- A client-exposed env variable that *isn't* meant to be world-readable.
- `===` or `.includes` comparing secrets without a constant-time function.
- An error response body that includes `err.message` for an error caught from an SDK call.

**All of these mean: rotate the secret if it shipped; fix the leak before the next commit.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "It's only the staging key" | Same discipline. Same boundary. |
| "We'll redact it later" | "Later" never happens once the log lands. Don't write it. |
| "Bundlers strip server code from client builds" | Yes — but client-exposed env vars are intentionally shipped. The rule is what *you* put in those vars. |
| "I'll use a short-lived token, it's fine to log" | Short-lived tokens are still credentials during their lifetime; an attacker with a recent log has a window. |
| "We have a WAF" | Defense in depth. A WAF doesn't help if the secret is in your own logs. |
| "Constant-time compare is paranoia at our scale" | Two extra lines. Zero timing-attack surface. Cheap insurance. |

## Reference

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) — canonical reference. Broader category: OWASP Top 10 A02:2021 (Cryptographic Failures).
- [Node.js `crypto.timingSafeEqual`](https://nodejs.org/api/crypto.html#cryptotimingsafeequala-b) — the constant-time comparison primitive used above.
