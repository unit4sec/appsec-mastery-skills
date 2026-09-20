---
name: "implement-app-attestation"
description: "Add app/device attestation so your SERVER gets hardware-backed, platform-signed proof that a request comes from your genuine, unmodified app on a genuine, untampered device (and, on Android, a licensed user). This is the server-side anchor that every on-device control (root detection, RASP, repackaging checks) ultimately depends on, because on-device checks are patchable. Android: Play Integrity API (verdict = app integrity PLAY_RECOGNIZED + device integrity MEETS_BASIC/DEVICE/STRONG_INTEGRITY + account/licensing), classic requests carry a nonce, standard requests carry a requestHash; the token is verified SERVER-side and bound to the request to stop replay. iOS: App Attest two-step model (one-time attestation binds a Secure Enclave key to a genuine app+device; per-request assertion signs each call; server verifies signature + nonce + a strictly increasing counter), plus DeviceCheck (2 bits of persistent per-device state for repeat-abuse limits). Always enforce server-side, require STRONG integrity for high-value actions, bind to fresh nonce/requestHash/counter, and plan humane fallbacks for legitimate failures. Use for banking, payments, anti-fraud, anti-cheat, and protecting any API from bots/emulators/modified apps."
---

# Implement App Attestation

Every on-device control is patchable because it runs on the attacker's device. Attestation moves the
trust decision off the device: the platform (Google/Apple), backed by the device's secure hardware,
produces a signed verdict that YOUR SERVER verifies. It answers, for the server: is this my genuine,
unmodified app? a genuine, untampered device? (and on Android) a licensed user? The app only carries
the token; verification and the decision happen server-side.

## 0. Clarify before coding
- What actions must be gated (login, payment, transfer, high-value API calls, promo/trial claims)?
- Required assurance per action (which map to STRONG vs lower integrity)?
- Do you have a backend to verify tokens/assertions and hold nonces/counters? (Required.)
- Android, iOS, or both? Fallback plan for devices that legitimately cannot attest?

## 1. Android — Play Integrity API
- **Verdict fields:** app integrity (`PLAY_RECOGNIZED` = your genuine, Play-signed app), device
  integrity (`MEETS_BASIC_INTEGRITY` weak / `MEETS_DEVICE_INTEGRITY` Google-certified OS /
  `MEETS_STRONG_INTEGRITY` hardware-backed), and account/licensing details.
- **Match the bar to the stakes:** require STRONG integrity for high-value actions; a lower bar avoids
  excluding legitimate users on older/non-certified devices for low-risk screens.
- **Request types:** *classic* carries a server **nonce** (you verify it round-trips in the token; do
  NOT cache classic verdicts); *standard* carries a **requestHash** (a digest of the request that you
  recompute server-side and compare), with Google Play adding on-device caching + replay protections.
- **Verify SERVER-side:** decode/verify the signed token with Google, read the verdict, confirm the
  nonce/requestHash matches THIS request, then decide allow / step-up / deny.

## 2. iOS — App Attest (two-step)
- **Attestation (once per install):** the app generates a key whose private half lives in the **Secure
  Enclave** and never leaves; `DCAppAttestService.attestKey(keyId, clientDataHash:)` with your server
  challenge returns an attestation object. Your server verifies it **once** and stores the public key.
- **Assertion (per request):** `DCAppAttestService.generateAssertion(keyId, clientDataHash:)` signs the
  request (plus a server nonce) with the attested key. Your server verifies the signature with the
  stored public key, checks the nonce, and enforces a **strictly increasing counter** (reject if it
  does not increase) to stop replay.

## 3. iOS — DeviceCheck (the sibling)
- Two bits of server-side state that **persist per device across reinstalls**. Use for per-device fraud
  limits (e.g., one free trial/promo per device). App Attest proves genuineness; DeviceCheck remembers
  a device over time. (On Android, approximate this with the Play Integrity verdict + your own
  server-side accounting.)

## 4. Enforce it (or it is worthless)
- Attestation only helps if the **backend enforces** the verdict, the freshness binding
  (nonce/requestHash), and (iOS) the counter. Fetching a verdict and ignoring a bad one gains nothing
  (the detection-to-enforcement gap).
- Gate sensitive actions on the server-verified result; decide allow / step-up / deny on the server.

## 5. Respect the limits (strong, not magic)
- Prefer hardware-backed **STRONG** integrity for high-value actions; weaker levels are more spoofable.
- **Availability:** older devices, devices without Google Play services, and some enterprise devices
  may fail legitimately — plan a fallback and do not blanket-lock real users.
- Bypass attempts exist (e.g., Play Integrity Fix), so treat the verdict as a strong signal within your
  defense in depth, alongside the on-device layers (root detection, RASP, repackaging checks).

## Final checklist
- [ ] Sensitive actions gated on a SERVER-verified attestation result.
- [ ] Android: Play Integrity verdict checked (app + device integrity + account), level matched to stakes.
- [ ] Freshness binding: classic nonce verified, or standard requestHash recomputed and matched; no caching of classic verdicts.
- [ ] iOS: attestation verified once (public key stored); per-request assertion signature + nonce + increasing counter checked.
- [ ] DeviceCheck (or server-side accounting) used for per-device repeat-abuse limits where needed.
- [ ] STRONG integrity required for high-value actions; humane fallback for legitimate failures.
- [ ] Attestation layered with on-device controls, not treated as the only defense.

## Anti-patterns to refuse
- Reading a verdict/assertion on the client and trusting it without server verification.
- Skipping the nonce/requestHash/counter checks (replayable) or caching classic verdicts.
- Requiring STRONG integrity everywhere and hard-locking legitimate users who cannot attest.
- Treating attestation as a magic bit instead of enforcing it server-side within defense in depth.
- Storing App Attest keys anywhere but the Secure Enclave, or shipping secrets that attestation is meant to replace.
