---
name: "implement-rasp-hook-detection"
description: "Add runtime application self-protection (RASP) to a mobile app the honest way. RASP is the app defending itself from the inside WHILE it runs, complementing obfuscation (which fights static reading) by fighting hooking, debugging, and tampering at runtime. Detect dynamic instrumentation (Frida via memory maps/frida-agent, frida-server + ports 27042/27043 though any port is possible, threads gmain/gum-js-loop, D-Bus channel, inline hooks/patched prologues) and frameworks (Xposed, LSPosed, Zygisk, Cydia Substrate, SandHook). Detect debuggers (Android TracerPid in /proc/self/status, self-ptrace, iOS PT_DENY_ATTACH + sysctl P_TRACED, block JDWP, timing). Verify integrity at runtime (SHA-256 of DEX/APK/native libs, signing certificate, installer source, debuggable flag). Respond quietly: never crash at the check; vary + delay the response, degrade/wipe/block, and decide on the SERVER. RASP runs on the attacker's device so it is bypassable (even the checks can be hooked): make it count with native + obfuscated + layered checks anchored by server-side attestation. Use for banking, payments, anti-fraud, anti-cheat, and high-value apps."
---

# Implement RASP & Hook Detection

Obfuscation fights someone READING your code; RASP fights someone RUNNING it and hooking, debugging,
or tampering with it live. RASP is your app watching its own runtime from the inside and reacting to
attack in real time. It is a cost-raising layer and a real-time signal source, not an unbreakable
wall, so it only works as part of defense in depth anchored by server-side attestation.

## 0. Clarify before coding
- What are you protecting: payments, anti-fraud, anti-cheat, licensing, sensitive data?
- What is the response policy per risk: block, step-up, degrade, wipe, or just log/score server-side?
- Android, iOS, or both? Any cross-platform runtime (Flutter/RN) needing native checks?
- Do you have a backend risk engine to receive signals and make the final decision? (You need one.)

## 1. Detect hooking / dynamic instrumentation
- **Frida** signals: frida-agent library in the process memory maps; frida-server process and default
  ports 27042/27043 (treat ports as a weak hint, Frida can use any port); characteristic threads
  (gmain, gum-js-loop); its D-Bus-style channel; and **inline hooks** (function prologues rewritten
  with trampolines / patched bytes) on your own critical functions.
- **Frameworks**: Xposed, LSPosed, Zygisk, Cydia Substrate, SandHook artifacts.
- Combine signals; no single one is reliable alone.

## 2. Detect debuggers (anti-debugging)
- Android: read **TracerPid** in `/proc/self/status` (0 = no tracer); **self-attach with ptrace** so no
  other debugger can attach; block **JDWP**.
- iOS: **ptrace PT_DENY_ATTACH**, and check the **sysctl P_TRACED** flag.
- Watch for timing anomalies (single-stepping is much slower than normal execution).
- Treat a debugger with the same seriousness as a hook: same power to rewrite behavior.

## 3. Verify integrity / detect tampering
- Compute a runtime **SHA-256** over your code (DEX/APK) and native libraries; compare to known-good.
  Any changed byte reveals a repackaged/patched app.
- Verify the app's **signing certificate** (repackaging forces re-signing with the attacker's cert).
- Check the **installer source** and that the **debuggable** flag is off in production.
- Cover resources/assets too, not just code. (This bridges to the repackaging-prevention control.)

## 4. Respond correctly (the detection-to-enforcement gap)
- Detection is the easy half; quiet, effective **enforcement** is where apps fail.
- **Never crash instantly at the check** — it points the attacker at exactly what tripped.
- **Vary and delay** the response, and act away from the detection site so the cause is not obvious.
- Choose a response to fit the risk: silently degrade, wipe the session/secrets, or block the sensitive
  action.
- **Report the signal to your server** and make the real decision in your risk engine, off-device.

## 5. Make RASP actually count (the honest reality)
- RASP runs on the attacker's device, so it can be bypassed — Frida can even hook your RASP checks and
  make the Frida-detector report all clear. It is a cat-and-mouse game.
- Raise the cost and make it durable: implement checks in **native code**, **obfuscate** them (from the
  obfuscation control), and **layer many** independent checks so one bypass does not win.
- Anchor the real trust decision **off the device** with **server-verified attestation** (Play Integrity
  / App Attest). RASP provides real-time signals; the server makes the call the attacker cannot patch.

## 6. Use vetted implementations
- Open-source: **Talsec freeRASP** (Android/iOS/Flutter/RN). Commercial: **Talsec RASP+**, **Promon
  SHIELD**, **Guardsquare** (DexGuard/iXGuard include RASP), **Appdome**, **Zimperium**.
- Understanding the mechanics lets you evaluate and configure these rather than treating them as magic.

## Final checklist
- [ ] Hooking detection: Frida (maps, server/ports, threads, D-Bus, inline hooks) + frameworks.
- [ ] Anti-debugging: TracerPid, self-ptrace, iOS PT_DENY_ATTACH/sysctl, block JDWP, timing.
- [ ] Integrity: runtime SHA-256 of code/native libs, signing cert, installer source, debuggable flag.
- [ ] Response is quiet: varied, delayed, off-site; degrade/wipe/block; final decision server-side.
- [ ] Checks are native, obfuscated, and layered (redundant).
- [ ] Trust anchored by server-verified attestation; RASP treated as a signal, not the boundary.
- [ ] Built on a vetted RASP library rather than hand-rolled.

## Anti-patterns to refuse
- Relying on RASP alone as a security boundary.
- Managed-code-only checks that are trivially hooked, or a single check with no layering/redundancy.
- Unobfuscated detection code (found and removed instantly).
- Crashing immediately at the detection site (trains the attacker to patch that check).
- Detecting tampering but never enforcing, or enforcing only on-device with no server decision.
- Trusting Frida default-port checks as sufficient detection.
