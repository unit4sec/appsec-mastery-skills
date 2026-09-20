---
name: "implement-webauthn-passkeys"
description: "Implement WebAuthn / FIDO2 passkeys for phishing-resistant login: registration (attestation) and authentication (assertion) ceremonies with correct challenge, origin/RP ID, user-verification, and signature-counter checks. Use when adding passkeys, passwordless login, or a phishing-resistant second factor. Prefer a maintained WebAuthn library; never hand-roll the crypto/verification."
---

# Implement WebAuthn / Passkeys

You are adding WebAuthn (passkeys) for login or step-up. **Do not hand-roll the CBOR/COSE
parsing, attestation, or signature verification**: use a maintained server library and the
platform/browser APIs. Your job is to run the two ceremonies correctly and validate every field.

## 0. Clarify before coding
- Goal: passwordless login, MFA/step-up, or both.
- Discoverable credentials (passkeys, usernameless) vs non-resident (username first)?
- Authenticators allowed: platform (Face ID / Windows Hello), roaming (security keys), or both.
- User verification: `required` (biometric/PIN) vs `preferred`.
- Attestation policy: `none` (default, best privacy) vs `direct`/enterprise (device assurance).

## 1. Set the Relying Party correctly
- **RP ID** = your registrable domain (e.g. `example.com`), not a full URL. It governs where the
  credential works (subdomains included). Get this wrong and logins silently fail or over-scope.
- **Origin** allow-list = the exact HTTPS origin(s) your app runs on. Verified server-side on every ceremony.

## 2. Registration ceremony (attestation): server checks
- Issue a **cryptographically random, single-use challenge**; store it in the session with a short TTL.
- Set `pubKeyCredParams` to strong algs (e.g. ES256 / EdDSA), `user.id` = an opaque random handle
  (NOT email/username), `authenticatorSelection` (residentKey, userVerification), attestation policy.
- On response, verify: **challenge matches**, **origin** is allowed, **RP ID hash** matches, the
  **UV flag** if you required it, attestation (only if policy needs it), then **store the public key,
  credential ID, and initial signature counter**. Never store a private key (there isn't one to store).

## 3. Authentication ceremony (assertion): server checks
- Issue a **new random challenge**; optionally send `allowCredentials` (omit for usernameless/passkey).
- On response, look up the stored public key by **credential ID**, then verify:
  **signature** against the stored public key, **challenge**, **origin**, **RP ID hash**, **UV flag**,
  and the **signature counter** is greater than the stored value (update it; a repeat/decrease = possible clone).

## 4. Why each check exists (don't skip any)
- **Challenge** single-use → blocks replay of a captured response.
- **Origin + RP ID binding** → the signature is worthless on a look-alike/phishing domain.
- **User verification flag** → proves a human passed biometric/PIN, not just device presence.
- **Signature counter** → detects cloned authenticators (flag/lock on anomaly).

## 5. Passkeys & recovery (practical)
- Use **discoverable credentials** for usernameless passkey login; expect synced (cloud, roams) and
  device-bound (higher assurance) passkeys.
- **Enroll 2+ authenticators** and provide a safe recovery path: a single authenticator is a single
  point of failure/lockout. Don't fall back to SMS OTP as recovery (reintroduces phishing).

## 6. Use libraries (don't hand-roll)
- Server: `@simplewebauthn/server` (Node), `py_webauthn` (Python), `webauthn4j` (Java), Go `go-webauthn`.
- Client: `@simplewebauthn/browser` or the native `navigator.credentials` API.
- Or an IdP/passkey provider (Auth0, Okta, Descope, Hanko, Corbado, Stytch) if you don't run auth yourself.

## Final checklist
- [ ] RP ID = registrable domain; origin allow-list verified server-side.
- [ ] Random single-use challenge per ceremony, short TTL, session-bound.
- [ ] `user.id` is an opaque random handle (not PII).
- [ ] Registration verifies challenge/origin/RP ID (+UV, +attestation if required); stores public key + credential ID + counter.
- [ ] Authentication verifies signature, challenge, origin, RP ID, UV, and counter increment.
- [ ] Strong algs (ES256/EdDSA); attestation `none` unless policy needs more.
- [ ] Discoverable credentials for passkeys; 2+ authenticators + recovery path.
- [ ] Implemented via a maintained WebAuthn library: no hand-rolled COSE/attestation parsing.

## Anti-patterns to refuse
- Skipping origin/RP ID validation (breaks phishing resistance).
- Reusing or long-lived challenges (replayable).
- Ignoring the signature counter (misses cloned authenticators).
- Using email/username as `user.id`.
- SMS OTP as the recovery fallback.
- Hand-rolling attestation/signature verification instead of a vetted library.
