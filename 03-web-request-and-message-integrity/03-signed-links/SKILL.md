---
name: implement-signed-action-links
description: Build secure server-generated action links (email verification, password reset, magic login, invites, one-click confirms). Treats the link as a bearer credential with high entropy, short TTL, single use, purpose and account binding, safe redemption (GET renders, POST acts), and leak defenses. Use for any emailed/tokenized action URL. Get the five properties and the redeem-then-invalidate order right.
---

# Implement Signed / Dynamic Action Links

You are building a server-generated action link (verify email, reset password, magic login,
invite, confirm). **Treat the link as a bearer credential: whoever holds it is the user.**
Do not hand-roll token crypto; use a CSPRNG or a maintained HMAC/JWS library.

## 0. Clarify before coding
- Action + blast radius (unsubscribe is low; reset/magic-login is account takeover).
- Token design: **opaque** (server-stored) or **signed** (stateless HMAC/JWS)?
- Delivery channel (email) and the sensitivity (does it need an extra confirmation step?).

## 1. The five required properties (all of them)
1. **High entropy**: 128+ bits from a CSPRNG. Unguessable, not merely unique (never a sequential ID).
2. **Short TTL**: minutes for reset/login, not days. Expiry enforced server-side.
3. **Single use**: invalidate on first successful redemption.
4. **Purpose/audience binding**: a `reset` token must be rejected at a `login` endpoint.
5. **Account binding**: the token names exactly one user and can act for no one else.

## 2. Token design (pick one, know the tradeoff)
- **Opaque (stateful):** store a row `token -> {user, purpose, expiry, used}`. Redeem = look up + check.
  Revocation and single-use are trivial (flip `used` / delete). Needs storage + a lookup.
- **Signed (stateless):** HMAC or JWS carrying `{user, purpose, exp}`; verify the signature, read claims.
  No lookup, but single-use/revocation still needs a **nonce store or deny-list** (a valid signature
  stays valid until expiry). Statelessness is not free.

## 3. Redemption flow (order matters)
1. User requests the action. Mint a fresh high-entropy token, bind user + purpose + expiry.
2. Deliver the link over the trusted channel.
3. On click, a **GET renders a safe, side-effect-free page** (see bots below).
4. On explicit confirmation, a **POST** hits the server.
5. **Validate**: exists, not expired, not used, correct purpose, correct account.
6. **Perform** the action.
7. **Invalidate** the token (mark used / delete). One token, one redemption, then dead.

## 4. Do not let a bot trigger it
- Mail clients, chat apps, and security scanners auto-fetch links to build previews / scan malware.
  They will "click" a bare GET. **Never change state on a GET.** Put the state change behind a
  human-confirmed POST so a preview bot cannot burn the token or fire the action.

## 5. Stop token leakage
- **Referer header:** a landing page loading third-party assets can send the full URL (token included)
  as the referrer. Strip the token from the URL before loading assets and set a strict `Referrer-Policy`.
- **History / shared devices:** short TTL + single use bound the window.
- **Email is not fully trusted:** treat it as a hint; add confirmation for sensitive actions.
- **Logs/analytics:** never log the token; redact query strings.

## 6. Build the URL safely
- **Host-header injection:** never build the link from the request `Host` header (an attacker sets it to
  their domain). Build every link from a **configured base URL**.
- **Open redirect:** a `next` / `returnTo` parameter you redirect to blindly can send users off-site.
  Allow-list safe internal paths; reject absolute/off-site targets.
- Magic link vs OTP: a magic link trades a long, leakable URL for convenience; an OTP is a short-lived
  code the user types with nothing long to leak. Choose per sensitivity.

## Final checklist
- [ ] Token is 128+ bit CSPRNG (or signed), never a guessable/sequential ID.
- [ ] Short TTL; expiry enforced server-side.
- [ ] Single use; invalidated on first successful redemption.
- [ ] Purpose/audience binding and account binding checked on redeem.
- [ ] GET renders only; state change on a confirmed POST.
- [ ] Referrer-Policy set + token stripped before third-party assets; token never logged.
- [ ] Links built from a configured base URL (no Host header); redirects allow-listed.
- [ ] Signed tokens: a nonce/deny-list backs single-use and revocation.

## Anti-patterns to refuse
- Sequential/guessable tokens (`?id=1043`).
- Non-expiring or reusable links.
- Acting on a bare GET.
- Putting the token where it leaks (referrer, logs) with no mitigation.
- Building links from the request Host header.
- Blindly redirecting to a user-supplied `next` parameter.
- Treating a signed token as single-use without a nonce/deny-list.
