---
name: implement-keyboard-clipboard-security
description: Close two quiet mobile leaks — the keyboard cache and the shared clipboard — so secrets typed or copied by users do not end up where other apps or the next person can find them. Keyboard cache: predictive text/autocorrect learns what is typed into normal fields and later suggests it (e.g. a previous user's password). Fix by marking sensitive fields: Android use a password input type or textNoSuggestions; iOS set autocorrectionType = .no and isSecureTextEntry = true. Clipboard: anything copied is on a system-wide clipboard readable by other apps (and malware); modern Android even toasts when an app reads it. Fix: prefer platform autofill over a Copy button, flag sensitive clips (Android ClipDescription EXTRA_IS_SENSITIVE so the preview is hidden), clear the clipboard shortly after a sensitive copy, and never auto-copy a one-time passcode. Apply to passwords, card numbers, PINs, OTPs, and security-question answers. Maps to OWASP MASVS-STORAGE (keyboard cache / copy-paste).
---

# Implement Keyboard Cache & Clipboard Security

Two easy-to-miss leaks: the keyboard's predictive-text cache and the system clipboard. Both keep copies
of what users typed or copied, where other apps or the next person on the device can find them. The
fixes are small per-field settings, but they must be applied to every sensitive field.

## 0. Clarify before coding
- Which fields hold secrets: passwords, card numbers, PINs, OTPs, security answers?
- Do you offer any "Copy" buttons on sensitive values (OTP, account number, card)?
- Do you auto-copy an OTP to the clipboard anywhere (a common, risky convenience)?

## 1. Keyboard cache — the leak and the fix
- Leak: on a normal text field, predictive text/autocorrect learns the value and can suggest it later,
  exposing a previous user's password/card number on a shared or stolen phone.
- **Android fix:** set the field's input type to a password type (e.g. `textPassword`/`numberPassword`),
  or `textNoSuggestions`, so the keyboard does not learn or suggest it.
- **iOS fix:** set `autocorrectionType = .no` and `isSecureTextEntry = true` on the field.
- Apply to every sensitive field, not just the password.

## 2. Clipboard — the leak and the fix
- Leak: the clipboard is system-wide; any app (including malware) can read whatever is copied. Modern
  Android shows a toast when an app reads the clipboard.
- **Best:** do not copy secrets at all — prefer the platform **autofill** to move a password/code into
  the field instead of offering a Copy button.
- If copying is required:
  - **Android:** flag the clip as sensitive (`ClipDescription.EXTRA_IS_SENSITIVE` /
    `android.content.extra.IS_SENSITIVE`) before `setPrimaryClip` so the preview is hidden.
  - **Clear** the clipboard a short time after a sensitive copy so it does not linger.
  - **Never auto-copy an OTP** to the clipboard by default.

## 3. Verify
- Type a secret into each sensitive field, then check the keyboard offers no suggestion of it (and the
  keyboard cache/dictionary does not contain it).
- Copy a sensitive value and confirm it is flagged sensitive and cleared; confirm no auto-copy of OTPs.
- Map coverage to OWASP MASVS-STORAGE (keyboard cache and copy/paste tests).

## Final checklist
- [ ] Every sensitive field uses a password input type / no-suggestions (Android) or secure entry + no autocorrect (iOS).
- [ ] Prefer autofill over Copy buttons for passwords and codes.
- [ ] Copyable secrets flagged sensitive (Android) and cleared soon after copy.
- [ ] No auto-copy of one-time passcodes to the clipboard.
- [ ] Applied to passwords, card numbers, PINs, OTPs, and security answers.

## Anti-patterns to refuse
- Leaving sensitive fields as plain text fields (keyboard learns and suggests them).
- Auto-copying an OTP to the clipboard for convenience.
- Offering Copy on secrets without flagging them sensitive or clearing them.
- Assuming the clipboard is private — it is readable across the device.
