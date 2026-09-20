---
name: "implement-bot-management"
description: "Stop bots and abuse the modern way. Do NOT ship a puzzle CAPTCHA that every user must solve (AI solves reCAPTCHA v2 at ~100%, humans at 50-85%). Default to invisible, risk-based, attestation-backed verification (Cloudflare Turnstile / Private Access Tokens), keep a visible challenge only as a step-up for high risk, and ALWAYS verify the token server-side (siteverify). Layer device fingerprinting (network JA3/JA4, browser canvas/WebGL/audio, behavioral, session) into a risk score that drives allow / step-up / deny. Use for login, signup, password-reset, checkout, and other abuse-prone or session-less endpoints. Treat every signal as a signal, not proof, and respect GDPR/ePrivacy: security can rely on legitimate interest, marketing needs consent."
---

# Implement Bot Management, Modern CAPTCHA & Device Fingerprinting

Classic "solve this puzzle" CAPTCHA is broken: modern AI vision solves reCAPTCHA v2 image
challenges at roughly 100% (ETH Zurich, YOLOv8), bots succeed at ~99.8% while humans manage
only 50-85%, and CAPTCHA farms defeat any puzzle for pennies. Puzzles now punish real users
(and fail accessibility) while waving bots through. Modern practice: read invisible risk
signals in the background and add friction only to suspicious traffic. This skill implements
that, plus the device-fingerprinting layer that feeds the risk decision.

## 0. Clarify before coding
- Which endpoints: login, signup, password-reset, OTP, checkout, comment/submit, scraping-prone APIs?
- Are the critical ones session-less (pre-login)? Those need the strongest layering.
- Region / compliance: EU users (GDPR + ePrivacy) change the legal basis and data you may collect.
- Existing edge/CDN (Cloudflare, Akamai) or bot-management vendor already in place?

## 1. Do NOT ship a puzzle everyone must solve
- A puzzle a human can solve, AI can solve, usually faster and more accurately.
- Never gate every visitor with an image/text puzzle. Reserve any visible challenge for step-up only.
- Honeypot fields (invisible input a human never fills) are a cheap first filter, not a real defense.

## 2. Default to invisible, risk-based, attestation-backed verification
- **Cloudflare Turnstile** (recommended default): free, no puzzle, background proof-of-work plus
  Private Access Tokens (PAT). Privacy-friendly, minimal data.
- **Private Access Tokens (PAT)**: built on the Privacy Pass IETF standard; the device
  cryptographically vouches that a real user on a real device is present WITHOUT revealing identity.
  Apple platforms (iOS 16+/macOS 13+) satisfy it silently, so the challenge disappears.
- **reCAPTCHA v3**: returns a 0.0-1.0 score from behavioral telemetry, BUT sends that telemetry to
  Google (GDPR concern) and its free tier was cut to ~10,000 assessments/month in 2025. Prefer
  privacy-preserving options unless you already depend on it.
- Alternatives: hCaptcha (enterprise), Friendly Captcha (proof-of-work, GDPR-friendly), Arkose Labs
  (adversarial), GeeTest (adaptive). Full platforms: DataDome, HUMAN.

## 3. ALWAYS verify the token server-side (non-negotiable)
- The client returns a token; your server MUST call the provider's **siteverify** endpoint to validate
  it before trusting the request. The client can lie, so the client is never the judge.
- Check the returned success flag, the score/verdict, the action name, the hostname, and token freshness.
- Never make the allow/deny decision in client JavaScript.

## 4. Score-driven decision: allow / step-up / deny
- Combine signals into a single risk score, then branch:
  - **Low risk** -> allow, zero friction (most traffic is legitimate; do not punish it).
  - **Medium risk** -> step up: show a visible challenge or require extra authentication.
  - **High risk** -> deny / block.
- This is where a visible CAPTCHA legitimately belongs: the medium-risk step-up, not every request.

## 5. Device fingerprinting: the signal layers
Combine many signals into a stable-ish identifier that recognizes a returning device even without
cookies. Sort them by layer (checked earliest at the top):
- **Network** (before any JavaScript runs, hard to patch from the page): IP reputation and ASN,
  TLS fingerprint **JA3 / JA4** (hash of the ClientHello cipher/extension list and order), HTTP/2
  SETTINGS frame order. Datacenter ASNs look unlike home connections.
- **Browser** (hundreds of bits of entropy): canvas hash (varies by GPU/driver/fonts/OS), WebGL
  renderer string + shader quirks, Web Audio oscillator hash (a headless browser with no audio device
  stands out), font enumeration, screen properties, navigator coherence / headless tells.
- **Behavioral**: mouse trajectory (real paths curve, accelerate, overshoot; scripts jump straight to
  center), scroll momentum, keypress cadence variance, focus/blur timing.
- **Session**: cookie persistence and request cadence (humans pause and vary; machines fire uniform bursts).

## 6. Entropy vs stability
- Each signal adds bits of entropy (uniqueness), but fingerprints **drift** as browsers, drivers, and
  fonts update. Too many fragile signals produce a unique-but-unstable identifier that changes each visit.
- Weight durable signals more heavily; expect gradual drift. A fingerprint is **stable-ish, not permanent**.

## 7. Depth beats any single check
- Treat every fingerprint as a **signal, not proof**. Anti-detect browsers spoof canvas, renderer, and
  JA3 values. Any single check can be defeated by someone who studies it.
- Combine layers (network + browser + behavioral + attestation). Spoofing one convincingly is not enough
  when several must agree: spoof the canvas, and the mouse still moves like a script, or the ASN is a
  datacenter, or attestation fails.

## 8. Layer with rate limiting, not instead of it
- CAPTCHA / bot management is ONE signal. Keep rate limiting on the same endpoints (see the
  rate-limiting skill), especially session-less critical routes where IP is weak.

## 9. Privacy and legality (GDPR / ePrivacy)
- A device fingerprint is **personal data** (a pseudonymous identifier). The ePrivacy Directive requires
  consent to store or access information on a device, and regulators apply this to fingerprinting.
- **Purpose governs the legal basis**: fraud prevention / security can typically rely on legitimate
  interest (GDPR Art. 6(1)(f)) and may be strictly necessary for security. Marketing, analytics, or
  profiling require **prior consent**.
- Minimize collected data, document a legitimate-interest assessment, be transparent, and **never
  repurpose a security fingerprint for tracking / advertising**. Prefer server-side signals that the
  client cannot see or edit.

## Final checklist
- [ ] No puzzle gating every user; visible challenge is step-up only.
- [ ] Default path is invisible + attestation-backed (Turnstile / PAT) or a privacy-preserving equivalent.
- [ ] Token is ALWAYS verified server-side via siteverify (success, score, action, hostname, freshness).
- [ ] Risk score drives allow / step-up / deny; friction reserved for risky traffic.
- [ ] Fingerprint combines network, browser, behavioral, and session layers.
- [ ] Durable signals weighted higher; drift expected (stable-ish, not permanent).
- [ ] No single signal treated as proof; layers combined for defense in depth.
- [ ] Rate limiting layered on the same endpoints.
- [ ] GDPR/ePrivacy: security legal basis documented, data minimized, no repurposing for tracking.

## Anti-patterns to refuse
- Shipping a puzzle CAPTCHA that every visitor must solve.
- Trusting the client-side score/verdict without a server-side siteverify call.
- Treating one fingerprint signal (e.g. just canvas, or just IP) as proof of identity.
- Piling on fragile signals until the fingerprint changes every visit.
- Using CAPTCHA as a wall and dropping rate limiting.
- Reusing a security/fraud fingerprint for marketing or ad tracking, or fingerprinting for marketing
  without consent.
- Puzzle-only flows that break accessibility for blind, low-vision, or motor-impaired users.
