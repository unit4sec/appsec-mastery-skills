---
name: implement-oauth-securely
description: Implement OAuth 2.x delegated authorization securely in an app — Authorization Code + PKCE, exact-match redirects, state/iss checks, hardened token validation, and sender-constrained tokens. Use when adding "log in with…", third-party API access, or any OAuth/OIDC flow. Prefer configuring a trusted provider and a maintained library over hand-rolling.
---

# Implement OAuth Securely

You are implementing OAuth 2.x / OpenID Connect for an application. Follow this skill
exactly. **Do not hand-roll token parsing, signature verification, or the flow itself** —
use a maintained OAuth/OIDC library and a trusted authorization server. Your job is to
wire them correctly and enforce the checks below.

## 0. Clarify before coding
Ask (or infer from the repo) and state your assumptions:
- Client type: **web server (confidential)**, **SPA / mobile / desktop (public)**, or **machine-to-machine**.
- Provider: Keycloak, Auth0, Okta, Entra ID, Cognito, or a custom AS.
- Goal: authorization (access an API) vs authentication/login (**use OIDC**, not bare OAuth).
- Language/framework, and which maintained library you'll use.

## 1. Pick the right grant (2026)
- Interactive user (web/SPA/mobile/desktop) → **Authorization Code + PKCE (S256)**. Always.
- Machine-to-machine, no user → **Client Credentials**.
- **Never** use the Implicit grant or Resource Owner Password Credentials (deprecated; RFC 9700).

## 2. Authorization request — required parameters
- `response_type=code`
- `client_id`
- `redirect_uri` — pre-registered, **exact match** (no wildcards/prefixes/paths added at runtime).
- `scope` — **least privilege** only.
- `state` — cryptographically random, stored in the user's session; **compared on callback**.
- `nonce` — if using OIDC (ID token replay protection).
- `code_challenge` + `code_challenge_method=S256` — PKCE for **every** client, public and confidential.

## 3. Callback handling
- Reject if `state` is missing or doesn't match the stored value (CSRF/response-injection defense).
- If the AS returns `iss` (RFC 9207), verify it equals the expected issuer (mix-up defense).
- Exchange the `code` **on the back channel** (server-to-server) with the original `code_verifier`
  (+ client authentication for confidential clients). The access token must **never** appear in a
  front-channel URL.

## 4. Token validation (on every resource-server request)
Validate JWTs with a maintained library and an **explicit** policy — never trust the token's own header:
- **`algorithms` allow-list** (e.g. `["RS256"]` or `["ES256"]`). Reject `none` and any HMAC alg for
  asymmetric keys (prevents alg-confusion, CWE-347).
- **`iss`** equals a trusted issuer.
- **`aud`** names *this* resource server (prevents token confusion / confused deputy).
- **`exp`** (and `nbf`) within a small clock-skew leeway.
- Enforce required **scopes**/permissions per endpoint (object- and function-level authorization).
- Prefer verifying via the provider's **JWKS** endpoint with key caching + rotation.

## 5. Token hygiene & hardening
- **Access tokens short-lived** (minutes). **Refresh tokens rotate** on every use, with **reuse
  detection** (a replayed old refresh token revokes the whole token family).
- Prefer **sender-constrained tokens**: **DPoP** (RFC 9449) or **mTLS** (RFC 8705) so a stolen bearer
  token is useless to the thief.
- Bind tokens to a specific API with **Resource Indicators** (RFC 8707) / an `audience` parameter.
- Store tokens safely: web → httpOnly, Secure, SameSite cookies or server-side session; mobile →
  platform secure storage (Keychain / Keystore). Never in localStorage for a web SPA if avoidable.

## 6. Configure the provider (don't build an AS)
- **Keycloak:** per-client exact redirect URIs; *Proof Key for Code Exchange Code Challenge Method =
  S256*; disable Implicit/ROPC; enable refresh-token rotation; enable DPoP or mTLS client auth if available.
- **Auth0 / Okta / Entra / Cognito:** exact callback allow-list; require PKCE for public clients;
  refresh-token rotation + reuse detection; DPoP where supported; `audience`/`resource` per API.

## 7. Recommended libraries (don't hand-roll)
- Node/TS: `openid-client`. Python: `authlib`. Java/Spring: **Spring Security OAuth2 / Resource Server**.
  .NET: Microsoft.Identity / `Microsoft.AspNetCore.Authentication.JwtBearer`. Go: `coreos/go-oidc`.
- For SPAs: an AppAuth-style / provider SDK that does Authorization Code + PKCE. Not the implicit flow.

## Final checklist (verify before you ship)
- [ ] Authorization Code + PKCE (S256); Implicit/ROPC not used.
- [ ] `redirect_uri` exact-match, pre-registered.
- [ ] `state` sent and verified; `iss` verified when present.
- [ ] Code exchanged on the back channel; no token in any URL.
- [ ] JWT validated with algorithms allow-list + `iss` + `aud` + `exp`.
- [ ] Least-privilege scopes; per-endpoint authorization enforced.
- [ ] Short access tokens; refresh rotation + reuse detection.
- [ ] Sender-constrained (DPoP/mTLS) and/or audience-bound where feasible.
- [ ] Tokens stored in secure, non-JS-readable storage.
- [ ] Implemented via a maintained library + configured provider — no custom token verification.

## Anti-patterns to refuse
- Trusting the JWT `alg` header / using a public key as an HMAC secret.
- Wildcard or substring `redirect_uri` matching.
- Optional/skippable PKCE, or skipping `state`.
- Accepting a token without checking `aud`/`iss`.
- Putting access/refresh tokens in the URL or localStorage.
- Writing your own JWT verification instead of a vetted library.
