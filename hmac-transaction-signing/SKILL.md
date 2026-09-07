---
name: implement-request-signing
description: Sign and verify API requests, transactions, and webhooks with HMAC (or asymmetric signatures) — an unambiguous canonical string that covers method, path, sorted query, body hash, timestamp, and nonce; constant-time verification; replay protection; server-pinned algorithm; and key rotation via a KMS. Use for payment/transaction integrity, service-to-service auth, and webhook receivers. Never hand-roll the hashing; get the canonical string and check order right.
---

# Implement Request / Transaction Signing

You are adding message authentication to requests, transactions, or webhooks. **Never do
`H(secret ‖ message)`** (length-extension forgery) and **never hand-roll the comparison**.
Use a library's HMAC + a constant-time compare. Your job is the canonical string, the check
order, replay protection, and key management.

## 0. Choose the primitive
- Both endpoints are yours and can share a secret → **HMAC-SHA-256** (fast, symmetric).
- A specific party must be provable / third parties verify / legal non-repudiation → **asymmetric**
  signature (Ed25519 or RSA): private signs, public verifies.

## 1. Build an UNAMBIGUOUS canonical string (what you sign)
Cover everything an attacker could tamper with that changes meaning:
- HTTP **method** (uppercase), **normalized path**, **sorted query**, **SHA-256 of the raw body**,
  a **timestamp**, a single-use **nonce**, and a **key ID**.
- **The body must be covered** (via its hash). Signing only the URL lets an attacker rewrite the JSON
  body while the signature still validates.
- **Serialize injectively**: length-prefix each field (`3:abc`) or use a separator that cannot appear
  in the data. Otherwise concatenation collides (`"a"+"b|c"` == `"a|b"+"c"`), which is a forgery primitive.
- `tag = HMAC(secret, canonical_string)`.

## 2. Verify in this order (cheap checks before crypto)
1. **Timestamp** within a tolerance window (e.g. ±5 min) → else reject.
2. **Nonce** unused (single-use store, TTL = the window) → record it; replay → reject.
3. Look up the **secret by key ID**; **pin the algorithm server-side** (ignore any client `alg`).
4. **Rebuild** the canonical string from the received request and **recompute** the tag.
5. **Constant-time compare** (never `==`) → accept, else 401.

## 3. Replay protection
- Timestamp window bounds exposure; the single-use nonce stops reuse *inside* the window. Use both.
  (See the Nonce & Replay skill for the ledger details.)

## 4. Verification traps to avoid
- **Timing leak (CWE-208):** `==` returns early and leaks the tag byte-by-byte. Use a constant-time equal.
- **Algorithm downgrade:** never let the request pick the algorithm (`alg: none`/`md5`). Pin it on the server.
- **Body re-serialization:** capture the **raw body bytes** before any JSON parser runs; re-serialized JSON
  differs byte-for-byte and breaks verification.

## 5. Key management
- Put a **key ID** in every request so the server selects the right secret.
- **Rotate with an overlap window** (old + new both valid), on schedule and on suspected compromise.
- Store secrets in a **KMS / secret manager**, never in code or config. Verifier gets read-only; a
  dedicated job rotates.

## 6. Webhooks (you are the receiver)
- Verify the provider's signature over `timestamp + "." + raw_body`, constant-time compare, and enforce
  a timestamp tolerance to reject replayed old events. Dedupe on the event id. (Stripe-style.)

## Final checklist
- [ ] HMAC via a library; **no** `H(secret ‖ message)`.
- [ ] Canonical string covers method, path, sorted query, **body hash**, timestamp, nonce, key ID.
- [ ] Injective serialization (length-prefixed / safe separator).
- [ ] Order: timestamp → nonce → lookup+recompute → constant-time compare.
- [ ] Replay: timestamp window + single-use nonce.
- [ ] Constant-time compare; algorithm pinned server-side.
- [ ] Raw body captured before parsing (requests and webhooks).
- [ ] Key IDs + rotation with overlap; secrets in a KMS.

## Anti-patterns to refuse
- `H(secret ‖ message)` or any raw-hash MAC.
- Signing only the URL / leaving the body uncovered.
- Ambiguous concatenation of signed fields.
- `==` / non-constant-time tag comparison.
- Trusting a client-supplied algorithm.
- Re-serializing the body before verifying.
- Hardcoded signing secrets; no rotation path.
