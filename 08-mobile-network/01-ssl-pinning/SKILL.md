---
name: "implement-ssl-pinning"
description: "Add SSL/TLS certificate pinning so a mobile app trusts ONLY your server, not the phone's whole list of trusted certificate authorities — defeating interception proxies and mis-issued/user-added CAs. Pin the public key (SPKI), NOT the leaf certificate (which breaks on renewal), and ALWAYS include a backup pin plus an expiration and a shipping/rotation path or you will brick your users (the most common pinning incident). Android: prefer Network Security Config (declarative pin-set + backup, enforced per connection) or OkHttp CertificatePinner; never ship debug overrides. iOS: evaluate server trust in the URLSession delegate or use TrustKit. Know it is bypassable: pinning is client-side, so on a rooted/jailbroken device Frida/Objection can hook the trust check and force it to pass (this is why a proxy can fail at first then suddenly work). Harden by moving pin validation into native C/C++, layering RASP + root/jailbreak + signature/integrity detection, failing quietly and deciding server-side. Pin selectively for high-value apps; pinning guards the channel while attestation proves the client — use both."
---

# Implement SSL Pinning

Ordinary TLS trusts the phone's large set of certificate authorities, so a single mis-issued, rogue,
or user-added CA (or an interception proxy) can sit in the middle. Pinning narrows trust to only your
server's key, so anything else is rejected. It is a strong control for high-value apps, but it is
client-side and must be operated carefully.

## 0. Clarify before coding
- Which connections are high-value enough to justify pinning (do not pin everything reflexively)?
- Who controls the server certificate lifecycle, and can you manage backup pins + rotation?
- Android, iOS, or both? Do you also have RASP/root-detection/attestation to layer with?

## 1. Why pin
- Default TLS trust is only as strong as the weakest of hundreds of CAs; a bad or user-added CA, or a
  proxy whose CA the device trusts, enables a man-in-the-middle. Pinning makes the app reject any
  certificate/key that is not your pinned one, regardless of who signed it.

## 2. What to pin (get this right or you brick users)
- Pin the **SPKI (public key)**, not the leaf certificate. The certificate renews regularly; the key
  usually stays the same, so SPKI pinning survives renewal without an app update. Leaf pinning breaks
  on every renewal.
- **ALWAYS include a backup pin** (a second key you control), plus an expiration and a plan to ship new
  pins before the certificate changes. A single pin with no backup locks out all users on any key change.

## 3. Android
- **Preferred: Network Security Config** — a declarative file listing your pin-set and backup pins with
  an expiration; the platform enforces it on every connection.
- **Alternative: OkHttp CertificatePinner** — SPKI SHA-256 pins on your HTTP client, in code.
- Never ship debug overrides (test certificates) in a release build.

## 4. iOS
- Evaluate the server trust in the **URLSession authentication-challenge delegate** and compare the
  SPKI hash, cancelling on mismatch; or use **TrustKit** (SPKI pinning + backup pins + reporting).
- Pin the SPKI so certificate renewal needs no app update. Runs alongside App Transport Security.

## 5. Know it is bypassable
- Pinning is client-side. On a rooted/jailbroken device, **Frida/Objection hook the trust-evaluation
  function and force it to succeed** (SSL Kill Switch on iOS; `objection android sslpinning disable`).
  Repackaging (patching the config / injecting a user CA and re-signing) is another route.
- The classic "worked at first, then the proxy got in" is exactly a runtime hook flipping the check
  while the app runs. On a compromised device, client-side enforcement can always be flipped.

## 6. Harden it (make bypass expensive)
- Implement pin validation in **native C/C++** (far harder to locate and hook than managed code).
- Layer with **RASP** (Frida/hook detection), **root/jailbreak detection**, and **signature/integrity**
  checks (repackaging). Any one firing signals tampering.
- Respond quietly (no obvious crash at the check) and make the final decision server-side.

## 7. Operate it well and know its place
- Keep backup pins + expiration + a rotation/shipping path; the top real-world failure is bricking your
  own users, not a hack. Pin selectively for high-value endpoints.
- Add **Certificate Transparency** monitoring to detect mis-issued certificates for your domain.
- Pinning protects the **channel**; it does not prove the caller is your genuine app. **Attestation**
  (verified server-side) proves the client. Use both together.

## Final checklist
- [ ] Pin the SPKI (public key), never the leaf certificate.
- [ ] Backup pin(s) + expiration + a way to ship new pins before the cert changes.
- [ ] Android: Network Security Config (preferred) or OkHttp CertificatePinner; no debug overrides in release.
- [ ] iOS: URLSession trust evaluation or TrustKit; alongside ATS.
- [ ] Pin validation in native code; layered with RASP + root/jailbreak + signature/integrity checks.
- [ ] Quiet, server-side decision on tamper; not an obvious local crash.
- [ ] Used selectively for high-value connections; Certificate Transparency monitoring in place.
- [ ] Paired with server-verified attestation (channel + client).

## Anti-patterns to refuse
- Pinning the leaf certificate (breaks on renewal) or shipping with no backup pin (bricks users).
- Shipping debug/test certificate overrides in a release build.
- Relying on pinning alone on a rooted device with no RASP/native hardening.
- Treating pinning as proof of app identity (that is attestation's job).
- Pinning everything reflexively with no rotation plan.
