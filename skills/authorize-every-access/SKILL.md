---
name: authorize-every-access
description: Use when a handler loads a resource by an ID from the request (`params.id`, `body.orderId`, a slug) and returns or mutates it. Use when a query filters by the resource's own ID but not by the caller's tenant/owner. Use when authorization is enforced in the UI, middleware, or a gateway but not in the handler that touches the data. Use when a new endpoint defaults to reachable without an explicit access check. Use when server code follows a user-supplied URL, hostname, or file path.
---

# Authorize Every Access

## Overview

**Every request gets an explicit, server-side authorization decision — *this* principal, acting on *this* object, deny by default — and wherever the policy can be expressed as data, the scope lives in the query itself.** Shape validation proves the input is well-formed; it never proves the caller is allowed. Those are different questions, answered in different places.

An authenticated request is not an authorized one. Knowing *who* is calling (authentication) says nothing about *what they may touch* (authorization). The gap between them is the most common breach class in production software.

## The Iron Rule

```
NEVER load or mutate a resource from a request without an explicit, server-side
authorization decision for THIS principal on THIS object — deny by default.
```

**No exceptions:**
- Not for "the UI already hides that button"
- Not for "middleware checked the token"
- Not for "the ID is a random UUID, nobody will guess it"
- Not for "it's an internal endpoint"
- Not for "it's read-only"

**The mechanism is a strong default, not part of the invariant.** Prefer enforcing the decision *in the query* — ownership in the `WHERE` clause — because fetch-then-check leaves the row in memory before the check runs. But fetch-then-authorize is a legitimate shape where the policy can't be expressed as a filter (a policy engine, a shared-with list, attribute-based rules). And *ownership* is one authorization model among several — a public post fetched by slug or a role-gated admin read is authorized by a different policy, still explicit, still deny-by-default. What is never optional is the decision.

## Why

The canonical failure is IDOR — Insecure Direct Object Reference, the core of OWASP's #1 category, Broken Access Control. A handler reads an ID from the request and fetches the row scoped to *nothing*:

```ts
// ❌ Broken object-level authorization.
// The query filters by the order's own id, not by who's asking.
export async function getOrder(req: Request, session: Session) {
  const { id } = orderParams.parse(req.params); // shape is valid...
  const order = await db.query.orders.findFirst({ where: eq(orders.id, id) });
  //                                                   ^ any authenticated user, any id
  return Response.json(order);
}
```

The input is perfectly well-formed — `parse-dont-validate` did its job. But `GET /orders/8a3f...` returns *whoever's* order has that ID. Change the ID, read another tenant's data. A random UUID is not access control: IDs leak through referrer headers, logs, shared links, and support tickets, and "hard to guess" is a probabilistic hope, not a boundary.

The fix is to make ownership part of the query itself, so an unauthorized row simply isn't returned — there is no code path that fetches first and checks later:

```ts
// ✅ Ownership is in the WHERE clause. A row the caller doesn't own
//    is never fetched — the missing case is "not found", indistinguishable
//    from "doesn't exist" (don't leak existence to non-owners).
export async function getOrder(req: Request, session: Session) {
  const { id } = orderParams.parse(req.params);
  const order = await db.query.orders.findFirst({
    where: and(eq(orders.id, id), eq(orders.orgId, session.orgId)),
  });
  if (!order) return Response.json({ error: 'not_found' }, { status: 404 });
  return Response.json(order);
}
```

Enforcing in the query, not in a separate `if`, matters: a fetch-then-check leaves a window where the wrong row is already in memory (and often already logged, or already returned in an error's `cause`). Scoping the query closes the window structurally. Where the policy genuinely can't live in a `WHERE` clause, fetch-then-authorize is the correct shape — centralize it, and keep the deny response indistinguishable from not-found.

## Detection

You are violating the rule if any of these are true:

- A query filters by the resource's own ID but not by the caller's `orgId` / `ownerId` / `tenantId`.
- The only access check for an endpoint lives in the client, in a route-group middleware, or in an API gateway — not in the handler that reads or writes the data.
- Authorization is checked *after* the resource is loaded (`const x = await find(id); if (x.ownerId !== me) throw`) when the policy could have been the query's `WHERE` clause — a smell, not always a violation: fetch-then-authorize is legitimate for policies a filter can't express.
- A new route is reachable without an explicit, positive access decision — the default is "allowed" rather than "denied."
- A mutating handler (`PATCH`, `DELETE`) trusts an ownership field from the request body instead of deriving it from the session.
- Server code issues an outbound request to a user-supplied URL/host without an allowlist (SSRF — now classified under Broken Access Control).

## The Pattern

### Deny by default: the access decision is positive, not the absence of a block

A route is authorized because something *granted* it, not because nothing *denied* it. Fail closed: if the check can't run (session missing, policy store down, an unmapped action), the answer is "no."

```ts
// ❌ Allow by default — a new action nobody added to the deny-list is open.
function canDelete(role: string): boolean {
  if (role === 'viewer') return false; // blocklist
  return true;                          // everything else, including unknown roles, allowed
}

// ✅ Deny by default — a new role or action is denied until explicitly granted.
const DELETE_ROLES = new Set<Role>(['owner', 'admin']);
function canDelete(role: Role): boolean {
  return DELETE_ROLES.has(role); // unknown/new role → false
}
```

Pair this with `exhaustive-switch` where the decision is over a closed set of actions: an unhandled case should fail closed, not fall through to permit.

### Derive the owner from the session, never from the request

The request says *what* to act on; the *session* says who is acting. Never let the body assert its own ownership.

```ts
// ❌ The body claims which org it belongs to — an attacker sets it to yours.
export async function createInvoice(req: Request, session: Session) {
  const input = invoiceInput.parse(await req.json()); // includes input.orgId
  return db.insert(invoices).values(input);           // trusts caller-supplied orgId
}

// ✅ Ownership comes from the authenticated session; the body carries data only.
export async function createInvoice(req: Request, session: Session) {
  const input = invoiceInput.parse(await req.json()); // no orgId field in the schema
  return db.insert(invoices).values({ ...input, orgId: session.orgId });
}
```

### Scoped IDs make the missing check visible

When identifiers are branded (see `branded-ids`), a query helper can *demand* the tenant scope in its type, turning "forgot the ownership filter" into a compile error rather than a runtime leak.

```ts
// A repository helper that cannot be called without a scope.
function findOrderForOrg(id: OrderId, orgId: OrgId): Promise<Order | undefined> {
  return db.query.orders.findFirst({
    where: and(eq(orders.id, id), eq(orders.orgId, orgId)),
  });
}
// There is no findOrder(id) that skips the scope — the unsafe call doesn't exist.
```

### Allowlist outbound targets (SSRF)

Following a user-supplied URL lets an attacker reach internal services, cloud metadata endpoints, or `localhost`. Decide by allowlist, resolve-then-check, and deny by default.

```ts
// ❌ Fetches wherever the user points — reaches 169.254.169.254, internal hosts, file://
async function fetchAvatar(url: string) {
  return fetch(url);
}

// ✅ Only known hosts over https; everything else denied.
const ALLOWED_AVATAR_HOSTS = new Set(['images.example-cdn.com']);
async function fetchAvatar(rawUrl: string) {
  const url = new URL(rawUrl);
  if (url.protocol !== 'https:' || !ALLOWED_AVATAR_HOSTS.has(url.hostname)) {
    throw new Error('avatar host not allowed'); // deny by default; map to 400 at the boundary
  }
  return fetch(url);
}
```

## Pressure Resistance

### "The middleware already authenticated the request"

Authentication is not authorization. Middleware proves *who* is calling; it rarely knows *which specific row* this handler is about to touch. The object-level check belongs where the object is loaded. A gateway that checks "is logged in" does nothing against IDOR.

### "The UI never shows that resource to the wrong user"

The UI is not a security boundary — it's a suggestion. The API is called directly with `curl`, from a script, or from a second account. Every enforcement that matters is server-side, in the handler.

### "The ID is an unguessable UUID"

Unguessable is not unauthorized. UUIDs appear in URLs, browser history, referrer headers, logs, error trackers, and shared screenshots. Access control is a decision you make, not entropy you hope holds.

### "It's an internal-only endpoint"

"Internal" is a network assumption that fails the moment the endpoint is exposed, proxied, or reached via SSRF. Endpoints authorize their own access regardless of where the caller is assumed to sit.

### "Re-checking ownership everywhere is noise"

It isn't a separate check if it's the query's `WHERE` clause — it's one clause, and it's the difference between a scoped read and a data leak. Where it *does* feel repetitive, that's the signal to push the scope into a typed repository helper (above), not to drop it.

## Red Flags

- A `findFirst` / `findById` whose `where` names the resource ID but no owner/tenant column.
- An after-the-fetch ownership `if` for a policy that could have been the query's `WHERE` clause.
- An `orgId` / `userId` / `tenantId` read from `req.body` and used as the owner.
- A new route added with no explicit access decision — reachable by default.
- A blocklist (`if role === 'banned'`) where a new unlisted role defaults to allowed.
- `fetch(userProvidedUrl)` with no host allowlist.
- Access control present in middleware/UI but absent from the handler.

**All of these mean: move the decision into the handler, scope the query to the principal, and default to denied.**

## Common Rationalizations

| Excuse | Reality |
|---|---|
| "The token was already verified" | That's authentication. This is authorization — a different question, per object. |
| "The frontend enforces it" | The frontend is bypassed by `curl`. Enforce server-side. |
| "Nobody can guess the ID" | IDs leak constantly. Guessability is not a control. |
| "We check it in one central place" | Centralizing is good — *if* that place is on the data-access path. Middleware usually isn't. |
| "It's behind the VPN" | Network position isn't authorization; SSRF and misconfig routinely cross it. |
| "Adding the scope to every query is tedious" | Put the scope in a typed repository helper so the unscoped call can't be written. |

## Related

- `parse-dont-validate` — that skill proves the input's *shape*; this one proves the caller's *permission*. A parsed body is well-formed, not authorized.
- `branded-ids` — scoped/branded IDs let a repository helper demand the tenant in its type, making a missing ownership filter a compile error.
- `secrets-handling` — the sibling security discipline for credentials; both fail closed and never leak existence/values in errors.
- `exhaustive-switch` — an access decision over a closed set of actions should fail closed on the unhandled case, not fall through to permit.
- `fail-fast` — a check that cannot run (missing session, policy store down) denies, rather than proceeding on a default.

## Reference

- [OWASP Top 10:2025 — A01 Broken Access Control](https://owasp.org/Top10/2025/A01_2025-Broken_Access_Control/) — still the #1 category; SSRF was merged into it in the 2025 edition as an access-control failure.
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — deny-by-default, enforce server-side, least privilege.
- [OWASP Insecure Direct Object Reference Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) — scope queries to the authorized owner rather than checking after the fetch.
