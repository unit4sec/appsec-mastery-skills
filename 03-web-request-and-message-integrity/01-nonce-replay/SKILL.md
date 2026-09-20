---
name: implement-replay-protection
description: Add replay protection to requests/messages so a captured, valid, signed request cannot be re-sent (CWE-294). Combines a timestamp skew window with a single-use nonce ledger, correct check order, and idempotency keys for safe retries. Use for payments, webhooks, OTPs, and signed APIs. Do the crypto with a maintained library; get the ORDER and SCOPE right.
---

# Implement Replay Protection

You are protecting an endpoint/message against replay: an attacker captures a valid,
signed request and sends it again. TLS and signatures do NOT stop this (they prove
confidentiality and authenticity, not **freshness**). Add a nonce + timestamp scheme.

## 0. Clarify before coding
- What is being protected: payment/transfer, webhook receiver, OTP, or signed API request?
- Do you control both sides (can add a nonce), or only the receiver (rely on provider's id + timestamp)?
- Where is the nonce ledger stored (Redis/DB) and what is the accepted clock-skew window?

## 1. What the client sends
- A **nonce**: cryptographically random, unique per request, unpredictable.
- A **timestamp**.
- A **signature** that covers the WHOLE payload **including the nonce and timestamp** (so an attacker
  can't swap in a fresh nonce). Reuse your request-signing scheme (HMAC/asymmetric).

## 2. Server check order (do NOT reorder)
1. **Timestamp**: reject if outside the skew window (e.g. ±5 min). Bounds how long you must remember nonces.
2. **Signature**: verify over the payload incl. nonce + timestamp (constant-time compare). Reject if invalid.
3. **Nonce ledger**: if the nonce is already present, reject (replay).
4. **Process** the request (do the work).
5. **Commit** the nonce to the ledger **only after success**, with **TTL = the skew window**.

Rationale: timestamp bounds the cache size; the nonce stops replay *inside* the window. Commit last so
a failed/forged request cannot burn a legitimate nonce or poison the ledger.

## 3. Idempotency keys (cooperative retries)
- For client retries after timeouts, accept a client-supplied **Idempotency-Key**. First time: do the
  work and store the result keyed by it. Same key again: return the **stored** result, do not repeat the action.
- Same foundation as replay defense ("remember what you've already done"), but for a *cooperative* duplicate.

## 4. Scope, storage, concurrency
- **Scope** the nonce correctly (per-key/tenant vs global) to match your trust boundary.
- Ledger in a shared store (Redis `SET NX` with TTL, or a unique DB constraint) so it works across instances.
- Make **check-then-commit atomic** (e.g. `SET key val NX EX <ttl>` returns whether it was new) to avoid a
  race where two copies slip through.
- Keep clocks synced (NTP) and the window tight.

## 5. Provider specifics
- **Webhooks (Stripe-style):** verify the signature AND enforce the event id + timestamp; dedupe on event id.
- **OTP / one-time codes:** single-use, short expiry, burn on first successful use.
- **AWS-style signed requests:** timestamp + signature are built in; add a nonce/dedupe if you need strict once-only.

> Not to be confused with a **CSP nonce**, which is a per-response token that whitelists inline scripts and
> has nothing to do with replay defense.

## Final checklist
- [ ] Nonce is random, unique, and **covered by the signature** (with the timestamp).
- [ ] Order: timestamp window → verify signature → check ledger → process → commit.
- [ ] Nonce committed **only after success**, TTL = skew window.
- [ ] Ledger is shared + atomic check-then-commit (no race).
- [ ] Correct nonce scope; tight window; synced clocks.
- [ ] Idempotency keys for safe client retries where relevant.
- [ ] Crypto via a maintained library (no hand-rolled signature/compare).

## Anti-patterns to refuse
- Relying on TLS or a signature alone for replay protection.
- Committing the nonce before verification/success.
- A nonce not included in the signed payload.
- A skew window so wide it bloats the ledger and lengthens the replay gap.
- Non-atomic check-then-commit (races let duplicates through).
