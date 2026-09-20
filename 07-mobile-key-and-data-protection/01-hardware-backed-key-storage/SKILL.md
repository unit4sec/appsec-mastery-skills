---
name: implement-hardware-backed-key-storage
description: Store and use cryptographic keys in secure hardware so they cannot be extracted even on a fully compromised (rooted/jailbroken) device. The principle across platforms: generate the key inside secure hardware, never let the private key leave, and use it by reference (sign/decrypt/key-agreement happen inside). Android: the Trusted Execution Environment (ARM TrustZone Secure World) via the Android Keystore for hardware-backed keys, and StrongBox (a dedicated tamper-resistant secure element, API 28+, setIsStrongBoxBacked) for the highest-value keys (fall back to TEE when absent). iOS/Apple silicon: the Secure Enclave (kSecAttrTokenIDSecureEnclave / CryptoKit SecureEnclave.P256), which supports only NIST P-256 keys and cannot export/sync. Bind keys to user authentication in hardware (Android setUserAuthenticationRequired/setUnlockedDeviceRequired; iOS kSecAccessControl with .biometryCurrentSet/.biometryAny). Prove hardware backing to your server with key attestation (verify the chain server-side). Because hardware keys are limited to certain algorithms/sizes, use them to wrap symmetric data keys (envelope encryption), not to hold bulk data. Use for any private key, device identity, payment/signing key, or secret you would otherwise store in software.
---

# Implement Hardware-Backed Key Storage

Any key your software can read can be stolen on a compromised device (files, backups, and process
memory are all readable by root/hooks/debuggers; obfuscation only delays it). The fix is architectural:
generate the key inside secure hardware, never let the private key leave, and use it by reference. This
control underpins device identity, payment/signing keys, and protecting other secrets.

## 0. Clarify before coding
- What key/secret are you protecting: device identity, payment/signing key, a symmetric data key?
- Assurance needed: is this a high-value key (favor StrongBox) or general use (TEE is excellent)?
- Should use require the user present (biometric/PIN) or the device unlocked?
- Does your server need to verify the key is truly hardware-backed (use key attestation)?

## 1. The cross-platform principle
- Generate the key **inside** secure hardware; the app receives a **handle/reference**, never the bytes.
- Sign / decrypt / key-agreement happen **inside** the secure world; only results come out.
- The private key is **non-exportable**. Never design a flow that needs to read the raw key.

## 2. Android — TEE (default hardware backing)
- Use the **Android Keystore** to generate keys; they live in the **TEE** (ARM TrustZone Secure World).
  Even a rooted OS cannot read them.
- Choose algorithms the hardware supports; do crypto through the Keystore key handle.

## 3. Android — StrongBox (highest assurance)
- **StrongBox** is a dedicated tamper-resistant secure element (API 28+, e.g. Titan-M class) with its own
  CPU/RAM/storage; it resists physical/hardware tampering, beyond what the TEE (which shares the SoC) does.
- Request it with `setIsStrongBoxBacked(true)`. It is slower and supports fewer algorithms/smaller keys,
  and is not on every device, so **fall back to the TEE gracefully** when unavailable.
- Use StrongBox for your highest-value keys; TEE for general use. Both beat software storage.

## 4. iOS / Apple silicon — Secure Enclave
- The **Secure Enclave** is a dedicated security coprocessor; keys are generated inside and never leave.
- Create keys with `kSecAttrTokenIDSecureEnclave` (or CryptoKit `SecureEnclave.P256`); reference them via
  the **Keychain** (which stores the handle and small secrets; use Data Protection classes for lock-state gating).
- **Constraints:** only NIST **P-256** EC keys (sign + key agreement); cannot store arbitrary data, cannot
  export, cannot sync via iCloud. Design around these (see envelope pattern below).

## 5. Bind key use to the user (hardware-enforced)
- **Android:** `setUserAuthenticationRequired(true)` (key usable only after biometric/PIN) and/or
  `setUnlockedDeviceRequired(true)`.
- **iOS:** `kSecAccessControl` requiring user presence; the ACL is evaluated **inside the Secure Enclave**.
  Prefer `.biometryCurrentSet` (key invalidated if biometrics change) over `.biometryAny` (survives
  enrollment changes, weaker) for sensitive keys.
- Because the check is in hardware, a hook cannot bypass it; a stolen key handle is useless without the live user.

## 6. Prove hardware backing to your server (attestation)
- Use **key attestation**: the hardware issues a certificate chain stating the key is hardware-backed and
  its properties (TEE vs StrongBox, user-auth required). Verify the chain **server-side** (up to the
  platform root of trust) before trusting the key. Do not trust the app's own claim.

## 7. Work within the limits (envelope pattern)
- Hardware keys support only certain algorithms and small sizes (e.g. Secure Enclave P-256 only). To
  protect bulk/symmetric data, **wrap a symmetric data key with the hardware key** (envelope encryption,
  from the payload-encryption control): the hardware protects the small key, the small key protects the data.
- Expect slower operations on StrongBox/Secure Enclave; keep hardware-key operations for what matters.

## Final checklist
- [ ] Keys generated inside secure hardware; private key non-exportable; used by reference.
- [ ] Android: hardware-backed Keystore (TEE); StrongBox requested for high-value keys with graceful fallback.
- [ ] iOS: Secure Enclave keys via Keychain; P-256 constraint respected.
- [ ] Key use bound to user auth / device unlock, enforced in hardware, where sensitive.
- [ ] Hardware backing proven via key attestation, verified server-side.
- [ ] Bulk/symmetric secrets protected by wrapping a data key with the hardware key (envelope).
- [ ] No secret kept in software that could live in hardware or on the server.

## Anti-patterns to refuse
- Storing a private key in a file, shared preferences, app memory long-term, or the app binary.
- Designing a flow that needs to export or read the raw hardware key.
- Trusting an app's claim of hardware backing without server-side key attestation.
- Requiring StrongBox with no TEE fallback (breaks on devices without it).
- Trying to store bulk data or unsupported key types in the Secure Enclave instead of wrapping a data key.
- Enforcing user-auth in app code instead of via the hardware key's authentication requirement.
