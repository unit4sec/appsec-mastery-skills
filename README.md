# AppSec Mastery — Implementation Skills

Ready-to-use **AI implementation skills** for the *Advanced Web & Mobile Application Security* course.

Each course topic ships one skill: a structured prompt you hand to your coding AI
(Claude Code, Cursor, Copilot, etc.) so it implements that security control
**correctly and securely** in *your* stack. The course teaches you the theory —
what the control is, the risks, and why each defense matters — and these skills let
you act on it without hand-rolling the dangerous parts.

## How to use

1. Open the topic folder (e.g. `oauth-security/`).
2. Copy the contents of `SKILL.md` into your AI coding assistant:
   - **Claude Code / Agent skills:** drop the folder into `.claude/skills/`.
   - **Cursor:** paste into a project rule, or `.cursor/rules`.
   - **Any chat-based AI:** paste `SKILL.md` as the system/first message, then describe your app.
3. Tell it your framework/language and let it implement — then review against the checklist in the skill.

> These skills encode current best practice (relevant RFCs, CWE mitigations, OWASP guidance).
> They favor **configuring a trusted provider and using maintained libraries** over custom crypto/auth code.

## Topics

| Topic | Skill | Course lectures |
|-------|-------|-----------------|
| OAuth Security | [`oauth-security/SKILL.md`](oauth-security/SKILL.md) | OAuth Parts 1–3 |
| Strong Customer Authentication | [`strong-customer-authentication/SKILL.md`](strong-customer-authentication/SKILL.md) | SCA Parts 1–2 |
| WebAuthn & FIDO (Passkeys) | [`webauthn-fido/SKILL.md`](webauthn-fido/SKILL.md) | WebAuthn Parts 1–2 |
| Nonce & Replay Protection | [`nonce-replay/SKILL.md`](nonce-replay/SKILL.md) | Nonce & Replay Attacks |
| HMAC & Transaction Signing | [`hmac-transaction-signing/SKILL.md`](hmac-transaction-signing/SKILL.md) | HMAC Parts 1–2 |
| Dynamic & Signed Action Links | [`signed-links/SKILL.md`](signed-links/SKILL.md) | Signed Links |
| Secure Object Access (IDOR/BOLA) | [`indirect-object-references/SKILL.md`](indirect-object-references/SKILL.md) | Indexed References Parts 1–2 |
| Rate Limiting | [`rate-limiting/SKILL.md`](rate-limiting/SKILL.md) | Rate Limiting |
| Bot Management, CAPTCHA & Fingerprinting | [`captcha-device-fingerprinting/SKILL.md`](captcha-device-fingerprinting/SKILL.md) | CAPTCHA Parts 1–2 |
| Mutual TLS (mTLS) | [`mutual-tls/SKILL.md`](mutual-tls/SKILL.md) | Mutual TLS Parts 1–2 |
| Payload & Field-Level Encryption | [`payload-field-encryption/SKILL.md`](payload-field-encryption/SKILL.md) | Payload Encryption Parts 1–2 |
| Root, Jailbreak & Emulator Detection _(Mobile)_ | [`root-jailbreak-emulator-detection/SKILL.md`](root-jailbreak-emulator-detection/SKILL.md) | Root/Jailbreak/Emulator Detection |
| Code Obfuscation _(Mobile)_ | [`code-obfuscation/SKILL.md`](code-obfuscation/SKILL.md) | Code Obfuscation |
| RASP & Hook Detection _(Mobile)_ | [`rasp-hook-detection/SKILL.md`](rasp-hook-detection/SKILL.md) | RASP Parts 1–2 |
| Repackaging Prevention _(Mobile)_ | [`repackaging-prevention/SKILL.md`](repackaging-prevention/SKILL.md) | Repackaging Prevention |
| App Attestation _(Mobile)_ | [`app-attestation/SKILL.md`](app-attestation/SKILL.md) | App Attestation Parts 1–2 |
| Hardware-Backed Key Storage _(Mobile)_ | [`hardware-backed-key-storage/SKILL.md`](hardware-backed-key-storage/SKILL.md) | TEE / StrongBox / Secure Enclave (Parts 1–3) |
| Static Sensitive Data Security _(Mobile)_ | [`static-sensitive-data/SKILL.md`](static-sensitive-data/SKILL.md) | Static Sensitive Data |
| SSL Pinning _(Mobile)_ | [`ssl-pinning/SKILL.md`](ssl-pinning/SKILL.md) | SSL Pinning Parts 1–2 |

_More topics are added lecture by lecture as the course grows._
