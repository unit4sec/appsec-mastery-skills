---
name: "implement-mutual-tls"
description: "Add mutual TLS (mTLS) so BOTH sides authenticate with certificates, not just the server. Use for service-to-service (microservices, zero trust, service mesh), B2B partner APIs, high-value financial APIs, and mobile/device identity (provision a client cert at activation, store the key in the secure keystore). mTLS proves WHO is connecting via proof of private-key possession (not replayable like a bearer token) and makes traffic hard to intercept with a MITM proxy; combine it with tokens/OAuth for WHAT they may do (RFC 8705 binds a token to the client cert). The hard part is the PKI lifecycle: issue, distribute, rotate, revoke; prefer short-lived certs and automate with a service mesh. NOT for the open consumer web. Authentication is not authorization."
---

# Implement Mutual TLS (mTLS)

Ordinary TLS authenticates only the server and encrypts the channel; the client stays anonymous at
the transport layer and proves itself later in the app (password/token). mTLS closes that gap: both
sides present a certificate, so each cryptographically proves its identity during the handshake,
before any application code runs. Use this to establish strong, non-replayable identity between
parties you control.

## 0. Clarify before coding
- Who are the two parties: service-to-service, B2B partners, mobile app to backend, or IoT devices?
- Is the caller set known and finite (good for mTLS) or the open public (use tokens instead)?
- Is there a CA / PKI already (internal CA, service mesh, cloud Private CA), or must one be set up?
- Will you terminate mTLS at the edge/LB or end-to-end at the service?

## 1. Only use mTLS where it fits
- Good fits: microservices / zero trust / service mesh, B2B partner APIs, high-value financial APIs,
  mobile and device identity.
- Bad fit: the open consumer web. You cannot distribute and maintain a client certificate for every
  anonymous visitor. Stay with tokens/passwords there.

## 2. Require and verify the client certificate correctly
- Configure the server/gateway to request AND require a client certificate (fail closed if absent).
- Verify ALL of: the full chain to a trusted CA, the validity window (not-before/not-after), and
  revocation (CRL or OCSP). Seeing "a cert exists" is not verification.
- The handshake's CertificateVerify proves the client possesses the matching private key; a stolen
  cert alone is useless without the key. Do not weaken this.

## 3. Trust only YOUR private CA for client auth
- Pin the trust anchor to your own private/internal CA. Do NOT accept certificates from public CAs
  for client authentication, or almost anyone can present a valid one.
- Keep server-trust (normal TLS) and client-trust (mTLS) trust stores separate and explicit.

## 4. Combine mTLS with tokens (identity vs permissions)
- mTLS is coarse: it proves WHO is connecting, not WHAT they may do. Do not use it as your only
  access control.
- Layer a token/OAuth for fine-grained scopes. Prefer **RFC 8705 certificate-bound access tokens**
  (sender-constrained): the token only works from the connection whose client cert it was bound to,
  so a stolen token cannot be replayed elsewhere.
- Authentication is not authorization: still enforce per-request permission checks.

## 5. Interception resistance (mobile especially)
- mTLS makes MITM interception hard: a proxy can present its own cert to the client, but to reach the
  real server it must ALSO present a valid client cert + private key it does not have, so the server
  rejects it.
- Mobile pattern: provision the client certificate at **activation/onboarding**, store the private key
  in the device secure keystore (iOS Keychain / Android Keystore), and use mTLS for all API calls.
- Caveat: if the cert + key can be extracted from the app, an attacker can load them into a proxy and
  bypass it. Pair mTLS with certificate pinning, secure key storage, and (where available) device
  attestation. It is a strong layer, not an absolute wall.

## 6. Own the PKI lifecycle (the real cost)
- **Issue** from a trusted CA. **Distribute** certs and keys to clients without leaking the keys.
  **Rotate** before expiry (or services suddenly stop talking). **Revoke** compromised certs
  immediately (CRL/OCSP).
- Prefer **short-lived certificates**: if a cert lives hours, a compromised one expires on its own,
  making revocation almost automatic (the SPIFFE/SPIRE model).
- Automate the whole lifecycle; do not manage certs by hand.

## 7. Budget the runtime cost
- The extra certificate exchange + verification make each new connection's handshake heavier: some
  added latency and CPU, worst with many short-lived connections.
- Amortize with keep-alive, connection pooling, and TLS session resumption so the full handshake
  happens rarely rather than per call.

## 8. Handle edge termination safely
- If you terminate mTLS at a load balancer / gateway, do NOT forward plain, unauthenticated HTTP to
  the backend. Propagate the verified client identity to the backend securely (e.g. a trusted header
  over an internal-only network, or re-establish mTLS internally).

## 9. Protect private keys
- Store private keys in a secure keystore, secrets manager, or HSM. Never commit them to a repo or
  bake them into a container image.

## Final checklist
- [ ] Used only where both parties are known/controlled (not the open consumer web).
- [ ] Server requires a client cert and fails closed when it is missing.
- [ ] Verifies full chain + validity + revocation (CRL/OCSP), not mere presence.
- [ ] Trusts ONLY your private CA for client authentication.
- [ ] Combined with token/OAuth for authorization; RFC 8705 binding where possible.
- [ ] Mobile: cert at activation, key in secure keystore, plus pinning/attestation.
- [ ] PKI lifecycle automated: issue, distribute, rotate, revoke; short-lived certs preferred.
- [ ] Runtime cost amortized with keep-alive / pooling / session resumption.
- [ ] Edge termination propagates verified identity; no plain HTTP to the backend.
- [ ] Private keys in keystore / secrets manager / HSM, never in repo or image.
- [ ] Authorization still enforced per request (authn is not authz).

## Anti-patterns to refuse
- Requesting a client cert but not verifying chain, validity, or revocation.
- Trusting public CAs for client authentication.
- Using mTLS as the only access control, with no per-request authorization.
- Terminating mTLS at the LB and forwarding unauthenticated HTTP to the backend.
- Long-lived client certs with no rotation or revocation path.
- Shipping the client cert + private key in the app with no pinning/attestation and assuming it is unbreakable.
- Private keys in source control or container images.
- Forcing mTLS on the open consumer web where certs cannot be distributed at scale.
