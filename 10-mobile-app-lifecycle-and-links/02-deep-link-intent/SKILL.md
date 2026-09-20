---
name: "implement-deep-link-security"
description: "Secure mobile deep links against hijacking. Old-style CUSTOM SCHEMES (myapp://) can be registered by any installed app, so a malicious app can intercept a link meant for yours — and anything inside the link (login tokens, OAuth codes, account data) is handed to whichever app receives it. Fix ownership by using VERIFIED links instead: Android App Links (android:autoVerify + a Digital Asset Links assetlinks.json on your HTTPS domain with the app's package + SHA-256 signing fingerprint) and iOS Universal Links (Associated Domains + apple-app-site-association) — only your app can open them. Never pass secrets in a link; for OAuth use App Links/Universal Links as the redirect plus PKCE so an intercepted code is useless. Treat all incoming deep-link data as untrusted input (validate it) and protect exported components/activities so another app cannot trigger them with a crafted intent. Watch App Links pitfalls (misconfigured assetlinks.json, wrong fingerprint, silent verification fallback to browser)."
---

# Implement Deep Link & Intent Hijacking Defense

Deep links open a specific screen in your app. The risk: an old-style custom scheme is not tied to you,
so any app can claim it and intercept links (and their contents) meant for your app — a common way to
steal login tokens and OAuth codes from magic links.

## 0. Clarify before coding
- Do any of your deep links carry sensitive data (tokens, codes, account identifiers)?
- Are you still using custom schemes (`myapp://`) for anything security-relevant?
- Do you control the HTTPS domain needed for verified links?
- Which components/activities are exported and reachable via links/intents?

## 1. The two problems
- **Ownership:** a custom scheme can be registered by ANY installed app, so a malicious app can receive
  your links first.
- **Exposure:** whatever is inside a link is handed to whichever app opens it — so a token/code in the
  URL leaks to a hijacking app.

## 2. Fix ownership — use verified links
- **Android App Links:** an HTTPS intent filter with `android:autoVerify="true"`, plus a Digital Asset
  Links file (`/.well-known/assetlinks.json`) on your domain listing the app's package name and SHA-256
  signing-cert fingerprint. Verified ⇒ only your app opens those links.
- **iOS Universal Links:** the Associated Domains entitlement + an `apple-app-site-association` file on
  your domain. Same effect: a malicious app cannot claim your domain.
- Stop using custom schemes for anything sensitive; a custom scheme cannot promise ownership.

## 3. Keep secrets out of links
- Never put tokens, one-time codes, or account secrets in a link's URL.
- For OAuth/magic-link sign-in: use a **verified redirect** (App Link / Universal Link) and **PKCE**, so
  an intercepted authorization code cannot be exchanged by an attacker.

## 4. Trust nothing that arrives
- Treat every parameter a deep link carries as **untrusted input**: validate type/range/format, and do
  not perform sensitive actions purely because a link asked.
- **Protect exported entry points** (Android exported activities/receivers/services; deep-link handlers):
  set `exported` deliberately, require permissions, and validate the caller/params so another app cannot
  drive them with a crafted intent (intent hijacking/redirection).

## 5. Watch verified-link pitfalls
- Misconfigured `assetlinks.json`, wrong SHA-256 fingerprint, or silent verification failure can make
  Android fall back to browser handling. Verify autoVerify actually succeeded in production builds.

## Final checklist
- [ ] Sensitive links use verified App Links / Universal Links, not custom schemes.
- [ ] Domain verification files published correctly (assetlinks.json / apple-app-site-association) with the right fingerprints.
- [ ] No tokens/secrets carried inside links.
- [ ] OAuth uses a verified redirect + PKCE.
- [ ] All incoming link data validated as untrusted; exported components protected.
- [ ] autoVerify success confirmed in production (no silent fallback).

## Anti-patterns to refuse
- Using a custom scheme (`myapp://`) as an OAuth redirect or for any sensitive link.
- Passing a token/code/secret inside a deep-link URL.
- Trusting deep-link parameters without validation, or exposing exported components without checks.
- Assuming App Links "just work" without verifying the domain file and signing fingerprint.
