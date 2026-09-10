---
name: implement-screenshot-screen-sharing-protection
description: Protect sensitive mobile screens from being captured or watched, as an anti-fraud control against surging screen-sharing scams and screen-watching banking malware (e.g. Hook, Hydra). If a fraudster or malware can see the screen, they can read one-time passcodes, balances, and account numbers and approve transfers. Block two leak types on your sensitive screens (login, OTP, balances, transfers, card details): saved captures (screenshots, screen recordings) and live views (screen sharing, mirroring, casting, remote access). Android has a real block via FLAG_SECURE (prevents screenshots/recording and shows black to a cast/screen-share; Android 14+ adds a screenshot-taken callback; you can also detect active projection). iOS cannot block capture of normal views, so use detect-and-hide: UIScreen.isCaptured + capturedDidChange to blur/hide sensitive content during capture/mirroring, userDidTakeScreenshot to react after, and a secure UITextField layer for the one truly excluded region. Apply only to sensitive screens (not the whole app), warn the user when sharing is detected, layer with RASP/accessibility-abuse detection, and know the limits (rooted bypass, a second camera).
---

# Implement Screenshot & Screen-Sharing Protection

This is an anti-fraud control. Screen-sharing scams (victims tricked into sharing their screen via
AnyDesk/TeamViewer/Zoom/Teams) and screen-watching banking malware (Hook, Hydra) let an attacker see
the login, the balance, and the one-time passcode, which is enough to commit fraud. Keep sensitive
screens from being captured or shown outside the app.

## 0. Clarify before coding
- Which screens are sensitive: login, OTP/passcode, balances, statements, transfer confirmation, card details?
- Android, iOS, or both? (The capabilities differ a lot.)
- Do you also have RASP / accessibility-abuse / remote-access detection to layer with?

## 1. Block two kinds of leak (on sensitive screens only)
- **Saved capture:** a screenshot or screen recording saved on the device (or grabbed by malware).
- **Live view:** screen sharing, mirroring, casting, or a remote-access session watching in the moment.
- Apply protection to sensitive screens only, not the whole app (users legitimately screenshot receipts, etc.).

## 2. Android — a real block
- Set **FLAG_SECURE** on the sensitive window/screen: screenshots and screen recording are blocked, and
  the screen shows **black** to anyone casting or screen-sharing.
- **Android 14+**: register a screenshot-taken callback to log/react when a screenshot occurs.
- Detect when **screen casting / MediaProjection** is active (a strong signal someone may be watching).
- Limits: weaker on rooted devices; cannot stop a second camera pointed at the screen. Layer other checks.

## 3. iOS — detect and hide
- iOS will **not** block capture of normal views. Use detect-and-hide:
  - **UIScreen.isCaptured** (+ the capture-changed notification): true during recording, mirroring,
    AirPlay, or screen sharing. When true, hide/blur the sensitive content so only a blur is seen.
  - **userDidTakeScreenshot**: fires *after* a screenshot; you cannot prevent it, but you can log or warn.
  - A **secure UITextField** layer is the one region iOS genuinely excludes from captures; use it for the
    most sensitive value if you need a true block.

## 4. Anti-fraud UX
- When you detect the screen is being shared/mirrored, **warn the user** that someone may be watching,
  and hide sensitive values (especially the OTP) until it stops.
- Never ask users to share their screen; reinforce that your bank/app never will.

## 5. Layer and respect limits
- Combine with **RASP** (remote-access / hook detection), **accessibility-abuse detection**, and
  root/jailbreak detection — the same malware that shares the screen often abuses accessibility.
- These controls raise the bar but are not absolute (rooted bypass, physical camera). Use defense in depth.

## Final checklist
- [ ] Sensitive screens identified (login, OTP, balances, transfers, card details).
- [ ] Android: FLAG_SECURE on those screens; screenshot callback (14+); projection/casting detection.
- [ ] iOS: isCaptured-driven hide/blur during capture; screenshot-after handling; secure-field for a true block.
- [ ] Protection scoped to sensitive screens only, not the whole app.
- [ ] User warned when screen sharing/mirroring is detected; OTP hidden while active.
- [ ] Layered with RASP / accessibility-abuse / root detection.

## Anti-patterns to refuse
- Assuming iOS can block screenshots of normal views (it cannot — detect and hide).
- Blanket-blocking the entire app (frustrates legitimate use).
- Treating FLAG_SECURE as absolute (rooted bypass; camera still works).
- Showing the OTP/sensitive data while a capture or screen-share is active.
- Relying on this alone without RASP / accessibility-abuse detection against screen-sharing malware.
