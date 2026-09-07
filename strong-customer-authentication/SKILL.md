---
name: implement-strong-customer-authentication
description: Implement Strong Customer Authentication (SCA) and dynamic linking for payments/high-risk actions: 2+ independent factors from different categories, phishing-resistant factors, transaction-bound approval (amount+payee signed), risk-based step-up, and 3-D Secure 2 integration. Use when building payment flows, PSD2-regulated actions, or high-value confirmations. Prefer a certified PSP/3DS provider + platform authenticators over hand-rolled OTP.
---

# Implement Strong Customer Authentication (SCA)

You are implementing SCA for a payment or high-risk action. Follow this skill. **Do not
hand-roll OTP/crypto or a 3-D Secure stack**: use a certified PSP/3DS provider and
platform authenticators (passkeys/FIDO2). Your job is to enforce the rules below and
bind approval to the transaction.

## 0. Clarify before coding
- Jurisdiction/regime (EU PSD2 or equivalent) and whether SCA is mandatory for this action.
- Action type: login vs **payment** (payments require **dynamic linking**).
- Client: web / mobile; available authenticators (passkey, platform biometrics, authenticator app).
- PSP / 3-D Secure 2 provider in use.

## 1. Two factors, different categories, independent
- Require **≥ 2 factors from ≥ 2 different categories**: knowledge (password/PIN), possession
  (passkey/security key/device), inherence (biometric). Two of the same category (password + PIN) is **not** SCA.
- Enforce **independence**: a single compromise must not defeat both factors. Don't let the password
  and the possession factor live behind the same single device unlock with no separation.
- **Avoid SMS OTP** as the possession factor (SIM-swap, SS7 interception, real-time phishing relay).
  Prefer **passkeys / FIDO2** (phishing-resistant, origin-bound) or an authenticator app; biometrics
  gated by a hardware-backed key.

## 2. Dynamic linking (mandatory for payments)
- **Show the payer the exact amount and payee** being authorized.
- **Bind** the authentication code/approval cryptographically to those values plus a fresh nonce:
  `code = sign(amount ‖ payee ‖ nonce ‖ timestamp)`. **What the user sees must equal what they sign.**
- Any change to amount or payee after approval **must invalidate** the code (server rejects on mismatch).
- Nonce/timestamp give replay protection; approvals are single-use and time-bounded.

## 3. Risk-based step-up
- Default to the least friction that's compliant; **escalate** strength as risk rises (new device, new
  payee, high amount, anomaly). Reserve full SCA + dynamic linking for risky/high-value actions.
- Compute risk from signals (device, history, velocity, geo). Fail toward *more* auth, not less.

## 4. 3-D Secure 2 (card payments)
- Integrate via your PSP: send rich risk data (device fingerprint, history, amount, payee) so the
  issuer **ACS** can choose **frictionless** vs **challenge**.
- On the challenge path, ensure the challenge carries **dynamic linking** (amount + payee bound).
- Handle both outcomes and the authentication result server-side; never trust a client-reported result.

## 5. Exemptions & liability (PSD2)
- SCA exemptions exist (low-value ~<€30 with cumulative caps; trusted beneficiary; **TRA** while fraud
  rate stays under thresholds: tighter thresholds for higher amounts). Apply them via your PSP.
- **An exemption reduces friction, not liability.** Track fraud rates; lose the exemption if they rise.

## 6. Use providers/libraries (don't hand-roll)
- AuthN: WebAuthn/passkeys (`@simplewebauthn/*`, platform APIs), or an IdP that supports FIDO2 + step-up.
- Payments/3DS2: a certified PSP (Stripe, Adyen, Checkout.com, Braintree, etc.): do not build a 3DS stack.
- Never build your own OTP delivery/verification as the primary factor.

## Final checklist
- [ ] ≥2 factors from different categories; independence preserved.
- [ ] No SMS OTP as the strong factor; phishing-resistant factor preferred.
- [ ] Payments: amount + payee shown AND signed (dynamic linking); tamper → reject.
- [ ] Approvals single-use, nonce + timestamp, server-verified.
- [ ] Risk-based step-up; fail toward more auth.
- [ ] 3-D Secure 2 via PSP; result validated server-side.
- [ ] Exemptions handled by PSP; fraud-rate monitoring in place.
- [ ] Implemented via certified provider + WebAuthn: no hand-rolled OTP/3DS/crypto.

## Anti-patterns to refuse
- SMS OTP as the primary possession factor.
- A generic OTP/challenge that doesn't show or bind the amount + payee.
- Two same-category factors (password + PIN) called "MFA".
- Trusting a client-reported 3DS/auth result.
- Reusable (non-nonce'd) approvals, or approvals not invalidated on transaction change.
- Building your own 3-D Secure or OTP crypto instead of a certified provider.
