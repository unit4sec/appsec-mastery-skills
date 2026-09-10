---
name: implement-overlay-accessibility-defense
description: Defend an Android app against the two tricks behind most banking-app takeovers — overlay/tapjacking and accessibility-service abuse (used by malware like Hook, Brokewell, ToxicPanda). An overlay draws a fake or invisible screen over your app so taps/inputs land where the user can't see (fake login capture, hidden confirm/permission grant). Defend by rejecting taps when obscured: setFilterTouchesWhenObscured(true) / android:filterTouchesWhenObscured (Android 12+ already blocks untrusted full-screen overlay taps by default), and disable sensitive actions when the window is covered, especially transfers, approvals, and permission prompts. Accessibility abuse turns Android's assistive API (read the whole screen + tap/type for the user) into remote device takeover that reads OTPs and approves transfers silently, even working around Android 13+ restricted settings; the platform (Play Protect, restricted settings, Google's crackdown) is the primary defense, and your app should detect a risky/unknown accessibility service active during sensitive actions and step up, hide secrets, or block. iOS is largely immune to both. Layer with RASP and attestation.
---

# Implement Overlay & Accessibility Defense

These are the two weapons behind most Android banking-app takeovers, and they are strongest together:
an overlay shows a fake screen or captures taps, while accessibility reads the screen and taps for the
attacker — enabling silent session attacks that read a one-time passcode and complete a transfer while
the victim sees nothing. Primarily an Android concern; iOS is largely immune.

## 0. Clarify before coding
- Which screens/actions are sensitive (login, OTP, transfers, approvals, permission prompts)?
- Do you already have RASP / root detection / attestation to layer with?
- Android only, or do you (mistakenly) plan iOS overlay defenses (not needed)?

## 1. Overlay / tapjacking — the attack
- A malicious app uses the "draw over other apps" permission to draw over your app: a fake login
  overlay that captures input, or an invisible layer so a tap lands on a hidden confirm/permission grant.
- Financial actions (transfers, approvals) are the top target — one hijacked tap can move money.

## 2. Overlay / tapjacking — the defense
- Turn on **filter touches when obscured** on sensitive views: `View.setFilterTouchesWhenObscured(true)`
  or `android:filterTouchesWhenObscured="true"`. Taps arriving through an untrusted overlay are ignored.
- **Android 12+** already blocks touches from untrusted full-screen overlays by default; keep the app
  targeting a modern SDK.
- On your most sensitive screens, **disable the action entirely when the window is obscured** (partially
  or fully) and warn the user that another app may be drawing over yours.
- Consider `setHideOverlayWindows(true)` for critical flows where supported.

## 3. Accessibility abuse — the attack
- Android's accessibility API can read the entire screen and perform taps/typing on the user's behalf
  (built for assistive tech). Abused, it enables full device takeover: read OTP/TOTP off the screen,
  auto-fill and approve transfers, and hide the activity (silent session attacks).
- Malware tricks users into enabling it (posing as an update/helper) and advanced families bypass
  Android 13+ restricted settings via custom loaders.

## 4. Accessibility abuse — the defense
- **The platform is the primary defense**: Play Protect, Android 13+ restricted settings (block
  sideloaded apps from enabling accessibility), and Google's Play-policy crackdown on misuse.
- **App-side**: during a sensitive action, detect whether a risky/unknown accessibility service is
  enabled/active. If so, step up verification, hide the secret (e.g. the OTP), or block the action.
- Combine with **Play Integrity / attestation** and **RASP** (the same malware often also hooks or shares
  the screen). Do not rely on accessibility detection alone.

## 5. Know the platform difference
- iOS is largely immune: apps cannot freely draw over other apps, nor wield accessibility for takeover
  the way Android allows. Focus these defenses on Android.

## Final checklist
- [ ] Sensitive views set filterTouchesWhenObscured; app targets Android 12+ for default overlay protection.
- [ ] Sensitive actions disabled when the window is obscured; user warned about draw-over apps.
- [ ] Detect a risky/unknown accessibility service active during sensitive actions; step up / hide / block.
- [ ] Rely on the platform (Play Protect, restricted settings) as primary accessibility defense.
- [ ] Layered with Play Integrity/attestation and RASP.
- [ ] Overlay/accessibility defenses scoped to Android (iOS largely immune).

## Anti-patterns to refuse
- Accepting taps on sensitive controls without filterTouchesWhenObscured / obscured-window checks.
- Showing the OTP or approving a transfer while the window is obscured or a risky accessibility service is active.
- Treating app-side accessibility detection as a complete defense (platform + attestation + RASP are needed).
- Assuming these are cross-platform problems (they are primarily Android).
