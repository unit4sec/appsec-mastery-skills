---
name: "implement-static-data-security"
description: "Store sensitive data at rest on a mobile device safely, assuming the attacker can own the device. Rule 1, minimize: do not store what you do not need (keep data server-side, prefer short-lived tokens). Rule 2, encrypt what you must keep with a hardware-backed key. Never store secrets in the clear in SharedPreferences/UserDefaults/plist, unencrypted SQLite/Room/Core Data, external/shared storage, logs, crash reports, the clipboard, caches, or cloud backups. Android (2026): EncryptedSharedPreferences/Jetpack Security is DEPRECATED — use DataStore for persistence + Google Tink for encryption + the Android Keystore for the key, SQLCipher for databases, and scope/disable backups. iOS: use the Keychain with a ThisDeviceOnly accessibility class (no iCloud sync), Data Protection classes for files, and wrap larger data keys with a Secure Enclave key; exclude sensitive files from backup. Maps to OWASP MASVS-STORAGE. Use for tokens, credentials, keys, PII, and financial data on device."
---

# Implement Static Sensitive Data Security

The mobile threat model assumes the attacker can fully own the device, so anything stored at rest in the
wrong place can be read. Two rules govern everything: (1) do not store what you do not need, and (2)
encrypt what you must keep with a hardware-backed key. This complements the hardware-key-storage control
(which protects the keys) by protecting the data.

## 0. Clarify before coding
- What sensitive data does the app actually persist (tokens, credentials, keys, PII, financial)?
- Can any of it stay server-side or be a short-lived token instead of stored locally?
- Android, iOS, or both? Any third-party SDKs / crash reporters that might capture data?

## 1. Rule 1 — minimize (the strongest control)
- Keep sensitive data server-side whenever possible; the safest data at rest is data you never wrote down.
- Prefer short-lived tokens (that expire) over long-lived secrets sitting on the device.
- Only persist locally what genuinely must be there; then apply Rule 2.

## 2. Never store secrets in the clear (the sinks to avoid)
- Plaintext **SharedPreferences / UserDefaults / plist**.
- Unencrypted **databases and files** (SQLite, Room, Core Data).
- **External / shared storage** readable by other apps.
- **Logs and crash reports** — never print secrets; scrub before logging and vet crash SDKs.
- The **clipboard**, cached network responses, and **WebView cache**.
- **Cloud backups** (Android auto-backup, iCloud) that carry secrets off-device.

## 3. Android — encrypt at rest (2026-correct)
- **EncryptedSharedPreferences / Jetpack Security Crypto is deprecated.** Do not build new code on it.
- Modern stack: **DataStore** for persistence + **Google Tink** for encryption (consistent, upgradeable)
  + the **Android Keystore** (hardware-backed, from the hardware-key control) to protect Tink's key.
- Encrypt structured data with **SQLCipher**; encrypt sensitive files before writing them.
- Control backups: set `android:allowBackup`/backup rules so secrets are excluded from cloud backups.

## 4. iOS — Keychain + Data Protection
- Store small secrets (tokens, passwords, keys) in the **Keychain**.
- Set accessibility to a **...ThisDeviceOnly** class so the secret is unlocked-only and **not synced to iCloud**.
- Use **Data Protection** (NSFileProtection) so file encryption is tied to the device lock state; choose the
  class matching how the data is accessed.
- For larger blobs, use **envelope encryption**: encrypt with a data key and wrap that data key with a
  **Secure Enclave** key.
- **Exclude sensitive files from backup**; never fall back to UserDefaults for secrets.

## 5. Verify
- Inspect on-device storage (files, DBs, prefs) on a test device and confirm no plaintext secrets.
- Confirm nothing sensitive appears in logs, crash reports, clipboard, or a device backup.
- Map coverage to **OWASP MASVS-STORAGE** / MASTG storage tests.

## Final checklist
- [ ] Data minimized: server-side where possible; short-lived tokens preferred.
- [ ] What is stored is encrypted at rest with a hardware-backed key.
- [ ] No plaintext secrets in prefs/UserDefaults/plist, plain DBs/files, or external storage.
- [ ] No secrets in logs, crash reports, clipboard, or caches.
- [ ] Secrets excluded from cloud backups (Android backup rules; iOS exclude-from-backup + ThisDeviceOnly).
- [ ] Android: DataStore + Tink + Keystore (not EncryptedSharedPreferences); SQLCipher for DBs.
- [ ] iOS: Keychain (ThisDeviceOnly) + Data Protection + Secure Enclave wrapping for larger data.

## Anti-patterns to refuse
- Storing tokens/keys/PII in plaintext SharedPreferences, UserDefaults, or a plist.
- Building new Android encryption on the deprecated EncryptedSharedPreferences.
- Logging secrets or letting a crash reporter capture them.
- Leaving secrets in the default (iCloud-synced) Keychain accessibility or in cloud backups.
- Putting sensitive data on external/shared storage, or in the clipboard for convenience.
- Trying to store bulk data in the Secure Enclave instead of wrapping a data key.
