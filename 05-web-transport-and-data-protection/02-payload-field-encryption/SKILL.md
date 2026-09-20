---
name: implement-payload-encryption
description: Encrypt sensitive data at the application layer, beyond TLS, so it stays protected after the TLS tunnel is terminated and at rest. Use field-level encryption for specific values (card number, SSN, health data) and full payload encryption for end-to-end secrecy of the whole body. Always use envelope (hybrid) encryption (a fresh random data key with AEAD like AES-256-GCM, the data key wrapped by a KMS/HSM master key or recipient public key), always AEAD (integrity, not just secrecy) with associated data (AAD) binding context, and rigorous key management (generate, distribute, rotate, revoke, separate; keys in a KMS/HSM, never in code or a client). Plan for the quantum "harvest now, decrypt later" threat with crypto-agility and hybrid ML-KEM key wrapping for long-lived secrets. Use for payments/banking APIs, health/PII data, multi-hop/message-queue architectures, mobile app payloads, and anything with a long confidentiality lifetime. Use vetted libraries (JWE, Google Tink, AWS Encryption SDK, libsodium); never roll your own crypto.
---

# Implement Payload & Field-Level Encryption

TLS protects data only in transit and only until it is terminated at the edge (load balancer,
gateway, CDN); after that the data is plaintext in logs, caches, queues, and databases. Payload and
field-level encryption seal the sensitive data itself at the application layer, so it stays protected
past TLS termination, across every hop, and at rest. This is defense in depth on top of TLS, not a
replacement for it.

## 0. Clarify before coding
- What data is sensitive, and what is its **confidentiality lifetime** (minutes, or years/decades)?
- Does only part of the message need protection (field-level) or the whole body (full payload)?
- How many hops / intermediaries (queues, brokers, third parties) touch the data after TLS ends?
- Is there a KMS/HSM already (AWS KMS, GCP KMS, Azure Key Vault, Vault)? A private CA / recipient keys?
- Compliance drivers: PCI DSS (card data), HIPAA (health), GDPR (PII)?

## 1. Choose field-level vs full payload
- **Field-level**: encrypt specific values (card number/PAN, SSN, health records) while the rest of the
  message stays readable for routing, indexing, and processing. Those values stay encrypted even in
  logs, caches, and backups. (See Mastercard/Visa message-level encryption, MongoDB CSFLE.)
- **Full payload**: encrypt the entire request/response body so only the intended recipient can open
  it; intermediaries move it but cannot read any field.
- Rule of thumb: field-level when only parts are sensitive and the rest must stay usable; full payload
  for true end-to-end secrecy.

## 2. Use envelope (hybrid) encryption
- For each message, generate a **fresh random symmetric data key**.
- Encrypt the payload with that data key using an **AEAD** cipher (AES-256-GCM or ChaCha20-Poly1305).
  This yields ciphertext + a unique nonce/IV + an authentication tag.
- **Wrap** the data key with the recipient's public key or a KMS/HSM master key. The sealed message
  carries: ciphertext, nonce, tag, and the wrapped data key.
- Hybrid gives you symmetric speed for the bulk data and asymmetric/KMS safety for key delivery.

## 3. Always AEAD + bind context with AAD
- Never encrypt without integrity. AEAD provides confidentiality **and** a tamper-detecting tag; any
  altered ciphertext fails to decrypt (no silent tampering).
- Put context in **Associated Data (AAD)** that is authenticated but not encrypted: bind recipient,
  purpose, and expiry so a valid ciphertext cannot be replayed in a different context.
- This is the same integrity thinking as HMAC/signing, built into the cipher.

## 4. Nail key management (the whole game)
- **Generation**: strong CSPRNG randomness. **Distribution**: never ship keys in the clear.
  **Rotation**: on a schedule and after any suspected leak. **Revocation**: retire a compromised key
  fast without losing access to already-protected data. **Separation**: per-recipient/per-tenant keys
  to limit blast radius.
- Keep keys in a **KMS/HSM**; the master key never leaves in plaintext. The app asks the KMS to
  wrap/unwrap per-message data keys; plaintext data keys live briefly in memory and are discarded.
- Never put a key in source code, config, an environment baked into an image, or a client/mobile app.

## 5. Plan for the quantum horizon
- **Harvest now, decrypt later (HNDL)**: adversaries passively record encrypted traffic today and
  decrypt it once a quantum computer can break today's key exchange. It is undetectable and believed
  already underway; it threatens data with a long confidentiality lifetime.
- **What breaks**: symmetric survives (Grover only halves strength, so **AES-256** stays ~128-bit and
  safe); **asymmetric key exchange/signatures (RSA, ECC) fall to Shor's algorithm**. In envelope
  encryption the vulnerable link is the **data-key wrapping/exchange**, not the bulk cipher.
- **Standards**: NIST finalized ML-KEM (FIPS 203, key exchange), ML-DSA (FIPS 204) and SLH-DSA
  (FIPS 205) for signatures in 2024; HQC added as a backup KEM in 2025. Hybrid PQC key exchange is
  already shipping in TLS 1.3.
- **Do now**: build **crypto-agility** (swap algorithms via config, not a rewrite); for long-secrecy
  data, wrap data keys with a **hybrid classical + ML-KEM** scheme; keep AES-256 for payloads;
  **inventory** where long-lived secrets live and how each is protected. Migration deadlines are near
  (major systems target ~2030).

## 6. Use vetted tools, never roll your own
- Payload standard: **JWE** (JSON Web Encryption) with **A256GCM** as the AEAD default.
- Libraries: **Google Tink**, **AWS Encryption SDK**, **libsodium** sealed boxes (implement envelope
  encryption correctly for you).
- Key management: AWS KMS, Google Cloud KMS, Azure Key Vault, HashiCorp Vault Transit.
- Field-level: Mastercard/Visa message encryption, MongoDB client-side field-level encryption.

## Final checklist
- [ ] Field-level vs full payload chosen deliberately per data sensitivity.
- [ ] Envelope encryption: fresh per-message data key, AEAD (AES-256-GCM), wrapped by KMS/recipient key.
- [ ] AEAD everywhere (never encryption without integrity); context bound in AAD.
- [ ] Unique nonce/IV per encryption; never reused with the same key.
- [ ] Keys in KMS/HSM; central rotation; per-recipient/tenant separation; nothing in code/config/client.
- [ ] Vetted library (JWE/Tink/AWS Encryption SDK/libsodium); no home-made crypto.
- [ ] Long-lived secrets: crypto-agile design + hybrid ML-KEM data-key wrapping; data inventory done.
- [ ] No plaintext logged before encryption.

## Anti-patterns to refuse
- Treating TLS as sufficient for data that must survive past the tunnel or live at rest.
- Rolling your own crypto, or using ECB / any non-AEAD mode.
- Reusing a GCM nonce with the same key (catastrophic: key-stream leak, forgery).
- Hardcoding an encryption key in a client/mobile app (extractable = not secret).
- Logging plaintext before encrypting it.
- Encrypting without binding context (no AAD), enabling ciphertext replay.
- Keys in source control, config files, or container images.
- Ignoring long-lived-secret exposure to harvest-now-decrypt-later; no crypto-agility plan.
