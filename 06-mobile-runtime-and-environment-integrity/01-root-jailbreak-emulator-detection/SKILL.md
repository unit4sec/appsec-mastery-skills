---
name: "implement-root-jailbreak-detection"
description: "Detect rooted/jailbroken devices and emulators the right way. Treat on-device checks (Android su/Magisk/build props/writable system; iOS Cydia/Sileo files, URL schemes, sandbox-escape, injected dylibs; emulator goldfish/ranchu/QEMU artifacts) as a SPEED BUMP that raises attacker cost, NOT as a security boundary, because anything running on the attacker's device can be hidden, patched, or hooked (Zygisk+DenyList+Shamiko, KernelSU, APatch; Frida, Objection, Liberty Lite). Move the trust anchor OFF the device with hardware-backed remote attestation verified SERVER-side: Play Integrity API on Android (SafetyNet is retired), App Attest / DeviceCheck on iOS, always binding a fresh one-time nonce to prevent replay. Assume the device may be fully compromised: keep secrets and sensitive logic server-side, layer obfuscation + RASP, and gate sensitive actions on the server-verified verdict. Use for banking, payments, anti-fraud, anti-cheat, and any high-value mobile app."
---

# Implement Root, Jailbreak & Emulator Detection

The mobile threat model is fundamentally different from the web: the attacker can own the device
AND your binary. They can read your app's files and memory, watch its traffic, repackage it, and hook
its code at runtime. Root/jailbreak removes the OS sandbox your app was implicitly trusting; an
emulator gives the attacker a fully observable, scriptable device for abuse at scale. Design every
control assuming the device may be fully compromised.

## 0. Clarify before coding
- What are you protecting: banking/payments, anti-fraud, anti-cheat, enterprise data, licensing?
- What is the response on a bad verdict: block, step-up auth, degrade features, or just log/score?
- Do you have a backend that can verify attestation and hold a nonce (you need one)?
- Android, iOS, or both? Minimum OS versions (affects attestation availability)?

## 1. Understand what on-device detection can and cannot do
- It is a SPEED BUMP: it raises attacker cost and filters low-effort abuse. It is NOT a wall.
- Anything running on the attacker's device can be hidden, patched, or hooked to report "clean".
- So never make a security decision that depends solely on a client-side "isRooted = false".

## 2. On-device signals (defense in depth, not the anchor)
- **Android root**: presence of the `su` binary or busybox in common paths; root manager packages
  (e.g. Magisk); dangerous build props (`test-keys`, `ro.debuggable=1`); unlocked bootloader / custom
  ROM indicators; system partition mounted writable or suspicious mounts.
- **iOS jailbreak**: jailbreak apps/files (Cydia, Sileo, known paths); suspicious URL schemes
  (`cydia://`); **sandbox-escape test** (can the app write outside its container? it must not); unexpected
  dynamic libraries injected into the process; symbolic links where real directories belong.
- **Emulator**: emulator build props (`generic`, `goldfish`, `ranchu`, `sdk`); QEMU/virtual-hardware
  artifacts; fake or too-perfect sensors/telephony; default/missing device IDs; absence of real-device
  traits (battery behavior, real GPU, sensor motion noise).
- Collect multiple signals; do not rely on a single check. Prefer native/obfuscated implementations.

## 3. Know how attackers bypass on-device checks (so you set expectations)
- **Android**: Zygisk + DenyList + Shamiko + the Play Integrity Fix module; KernelSU; APatch; Zygisk
  Assistant / ZygiskNext / NeoZygisk. They keep full root while apps see a "clean, certified" device.
- **iOS**: jailbreak-hiding tweaks (Shadow, Liberty Lite) and dynamic instrumentation (Frida,
  Objection) that hook the detection functions and force a clean result.

## 4. Move the trust anchor off the device (the real fix)
Use hardware-backed remote attestation, verified on YOUR server:
1. Server issues a fresh, **one-time nonce** (random challenge).
2. App requests attestation from the platform: **Play Integrity API** (Android) or **App Attest /
   DeviceCheck** (iOS).
3. Platform returns a **hardware-backed, signed verdict** covering the nonce.
4. App forwards the signed attestation to the server.
5. **Server verifies** the signature with Google/Apple and checks the verdict (device/app/account
   integrity) and nonce freshness.
6. Server decides: allow, step up, or deny. The decision lives on the server, never in a client value.

- Android note: **SafetyNet Attestation is fully retired (2025); use the Play Integrity API.**
- Enforce **nonce freshness + one-time use** server-side, or attackers replay a recorded valid
  attestation to fake a clean device. Bind the verdict to the current request/session.

## 5. Assume compromise (the mindset that ties it together)
- Keep real secrets and sensitive logic **server-side**; a rooted device can read anything in the app.
- Layer **obfuscation + runtime protection (RASP)** to slow hooking and patching.
- Combine layers: on-device checks (raise cost) + server-verified attestation (the anchor) + server-side
  secrets. No single control is sufficient.
- **Fail thoughtfully**: prefer gating sensitive actions, step-up auth, or feature degradation over a
  blunt crash; log the signal to your risk engine. Avoid alienating legitimate power users unless policy
  (e.g. banking/PCI) requires a hard block.

## Final checklist
- [ ] On-device checks treated as a speed bump, never the sole security decision.
- [ ] Multiple signals collected (Android/iOS/emulator), native and obfuscated where possible.
- [ ] Hardware-backed remote attestation in place (Play Integrity / App Attest / DeviceCheck).
- [ ] Attestation verified SERVER-side (signature + integrity verdict), decision made on the server.
- [ ] Fresh, one-time nonce bound to each attestation to prevent replay.
- [ ] Secrets and sensitive logic kept server-side; app assumes full device compromise.
- [ ] Obfuscation + RASP layered in.
- [ ] Thoughtful failure handling (gate/step-up/degrade + log), not just a crash.

## Anti-patterns to refuse
- Making access control depend on a client-side `isRooted`/`isJailbroken` boolean.
- Shipping only on-device detection and calling the app "protected".
- Using the retired SafetyNet Attestation API instead of Play Integrity.
- Accepting an attestation without server-side verification, or without a fresh one-time nonce (replayable).
- Storing secrets or critical business logic in the app expecting root to keep it safe.
- Hard-crashing on any root signal with no risk-based response and no logging.
