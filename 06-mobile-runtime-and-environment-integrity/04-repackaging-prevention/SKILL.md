---
name: implement-repackaging-prevention
description: Prevent app repackaging (an attacker decompiling your app, modifying it, re-signing with their own key, and redistributing it as a trojanized/cloned/cracked fake). The core on-device tell is that any modification forces re-signing, so verify your signing certificate at runtime against a known-good value; back it with a runtime integrity checksum (SHA-256 of code/resources), an installer-source check, and a debuggable-flag check. But on-device checks run on the attacker's device and can be patched out, so anchor recognition SERVER-SIDE: on Android the Play Integrity app-recognition verdict (PLAY_RECOGNIZED) plus the device-integrity verdict, on iOS App Attest/DeviceCheck plus App Store distribution and FairPlay. Practice distribution hygiene (Play App Signing, Code Transparency, official stores only) and respond to tampering quietly with the final decision made on the server. Use for banking, payments, wallets, and any branded/high-value app.
---

# Implement Repackaging Prevention

Repackaging is when an attacker takes your published app, modifies it, re-signs it with their own key,
and redistributes it, as a trojanized copy (malware/spyware/ad fraud injected), a clone (a fake
banking/wallet app that steals credentials), or a crack (license/payment/ad checks stripped). The
victim believes they installed your app; they installed the attacker's. This control makes that hard
to do undetected. It builds directly on the integrity checks from the RASP control.

## 0. Clarify before coding
- What is the risk if a fake of your app exists: credential theft, fraud, malware, brand damage, revenue loss?
- Android, iOS, or both? Are you on Play App Signing? Do you publish an App Bundle (for Code Transparency)?
- Do you have a backend that can verify a Play Integrity / App Attest token and make the final call?

## 1. Understand the attack (and its one weakness)
1. Attacker obtains and decompiles the published app (APK / IPA).
2. Modifies code or resources (inject, strip, or replace).
3. **Must re-sign** — any change invalidates your signature, and they cannot sign as you without your
   private key, so they sign with their own key. **This mandatory re-sign is your detection hook.**
4. Redistributes via third-party stores, sideloading, or phishing links.

## 2. On-device defenses (necessary, but patchable)
- **Signing-certificate verification (primary):** at runtime, read the app's own signing certificate and
  compare it to your known-good certificate/public-key hash. A mismatch means it was re-signed.
- **Integrity checksum:** runtime SHA-256 over code (DEX/APK), native libs, and key resources vs known-good.
- **Installer-source check:** confirm the app was installed from the official store, not sideloaded.
- **Debuggable-flag check:** production builds must never be debuggable.
- Do these in native + obfuscated code (from the obfuscation/RASP controls). Know the limit: they run on
  the attacker's device and can be patched out during repackaging.

## 3. Anchor recognition server-side (the decisive move)
- **Android — Play Integrity API app-recognition verdict:** `PLAY_RECOGNIZED` means this is your
  unmodified, Play-signed app; a repackaged/re-signed build comes back `UNRECOGNIZED_VERSION`. It is
  verified server-side with Google via a signed token, so a patched on-device check cannot forge it.
  Pair with the device-integrity verdict (MEETS_BASIC / DEVICE / STRONG_INTEGRITY).
- **iOS — App Attest / DeviceCheck:** server-verified app+device attestation; combined with App Store
  distribution and FairPlay encryption, which make iOS repackaging significantly harder to begin with.
- Same principle as root detection: the server, not the phone, decides whether the app is genuine.
  Bind the verdict to a fresh server nonce and verify it server-side.

## 4. Distribution hygiene
- Use **Play App Signing** so Google holds/guards your app signing key.
- Consider **Code Transparency** (app bundle) to prove your code was not altered after you built it,
  using a transparency key only you hold (separate from the Play-held signing key).
- Distribute only through official stores; treat third-party-store presence of your app as a red flag to monitor.

## 5. Respond correctly
- Do not crash at the check (it reveals what tripped). Vary and delay the response, degrade/wipe/block,
  and make the real decision on the server based on the attestation verdict.

## Final checklist
- [ ] Runtime signing-certificate verification against a known-good value.
- [ ] Integrity checksum (SHA-256) of code/native libs/resources.
- [ ] Installer-source and debuggable-flag checks.
- [ ] On-device checks implemented native + obfuscated.
- [ ] Server-side anchor: Play Integrity app-recognition (+ device integrity) / iOS App Attest, nonce-bound.
- [ ] Play App Signing enabled; Code Transparency considered; official-store-only distribution.
- [ ] Quiet, server-decided tamper response.

## Anti-patterns to refuse
- Relying only on an on-device signature check (patched out during repackaging).
- Shipping the signing key, or not using Play App Signing.
- Trusting a client-side "isGenuine" result with no server-verified attestation.
- Distributing via third-party stores / sideload as an official channel.
- Crashing at the detection site instead of a quiet, server-side decision.
- Production builds left debuggable.
