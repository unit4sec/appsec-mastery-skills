---
name: "secure-object-access"
description: "Reference objects safely and eliminate IDOR/BOLA by design. Prefer server-indexed access (the client never supplies a global ID); if a reference must be exposed make it opaque (UUID/ULID), keep it in the POST body not the URL, and for public/session-less access use an encrypted or signed, expiring, revocable token (presigned-URL pattern). Always authorize server-side. Use for any endpoint that takes an object id, \"my resources\" APIs, and shareable public links."
---

# Secure Object Access (eliminate IDOR/BOLA by design)

IDOR / BOLA (Broken Object Level Authorization, OWASP API Top 10 #1) happens when a client
supplies a global object id and the server acts on it without checking ownership. Patching
"authorize every endpoint" is fragile: one forgotten check is a breach. Design so the bug
cannot exist, then apply layered defenses when a reference must be exposed. **Always authorize
server-side regardless** of the layers below.

## Layer 1 (prefer): server-indexed access, no client-controlled global id
- The server presents the user's own objects and the client selects by a per-user **index** or
  opaque **handle**; the server maps it to the real object INSIDE the authenticated session.
- Use an **indirect reference map** (OWASP): a per-session table `handle -> real id`, populated
  ONLY with authorized values, never global, never shared across users.
- Design `/me/...` endpoints ("my resources") that resolve everything from the session, so no
  object id appears in the path. This removes the "forgot the check" failure class by construction.

## Layer 2: if a reference MUST be exposed, make it opaque
- Never expose sequential integer ids (enumeration + record-count leak). Use **UUIDv4** (122 random
  bits) or **ULID** (timestamp + random, better index locality) as the PUBLIC identifier.
- Keep the public identifier separate from the internal primary key.
- Opaque ids only resist enumeration; they are a weaker complementary layer, NOT authorization.

## Layer 3: keep references out of the URL
- An id/sensitive parameter in a GET query string leaks into browser history, server/proxy logs,
  and the Referer header (CWE-598). Send it in the **POST body** (or a header). OWASP ASVS: sensitive
  data never goes in the URL.

## Layer 4: public / session-less access = expiring, encrypted/signed capability
- With no session, the reference itself is the credential (a "capability URL"). It can leak, so it
  MUST expire. A short TTL turns a permanent leak into a temporary one.
- Issue an opaque token that **encrypts or signs** `{document ref, expiry, scope}`. The client sends
  it in the POST body; the server decrypts/verifies, checks expiry and scope, then serves. The client
  cannot read it or forge a later expiry / wider scope. This is the cloud presigned-URL pattern
  (AWS S3 presigned, CloudFront / GCS signed URLs).
- Capability hygiene: HTTPS only, easy revocation, short TTL, last-access monitoring, never log the token.

## Layer 5 (foundation): authorize server-side, every request
- Enforce object-level authorization through a single **centralized** access-control mechanism
  (OWASP ASVS), not scattered per-endpoint checks. Building the index/map is itself an authorization
  decision and must be enforced, not assumed.

## Final checklist
- [ ] Default to server-indexed access / indirect reference map / `/me` endpoints.
- [ ] No client-supplied global object id where avoidable.
- [ ] Exposed references are opaque (UUIDv4/ULID); public id separate from internal PK.
- [ ] References travel in the POST body/headers, never the URL.
- [ ] Public/session-less access uses an encrypted or signed, expiring, revocable token.
- [ ] TTL short; HTTPS only; revocation available; token never logged; last-access monitored.
- [ ] Object-level authorization enforced server-side on every request, centrally.

## Anti-patterns to refuse
- Trusting a client-supplied global id without an ownership check.
- Relying on "we authorize on every endpoint" with no centralized mechanism.
- Sequential/guessable public ids; exposing the internal primary key.
- Object ids in the URL/query string.
- Public links that never expire or cannot be revoked.
- Treating opaque ids or expiry as a substitute for authorization.
