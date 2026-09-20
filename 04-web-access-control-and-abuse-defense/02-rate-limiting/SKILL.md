---
name: implement-rate-limiting
description: Add rate limiting to an app the right way. Enforce it CENTRALLY at an API gateway or dedicated service (not scattered per microservice), use a token-bucket algorithm (not a naive counter) backed by a shared atomic store, key by session/account when authenticated and IP only pre-auth, return 429 + Retry-After, and require a CAPTCHA on critical session-less endpoints. Use for login/abuse-prone routes, expensive APIs, and DoS/cost protection. Complements auth, never replaces it.
---

# Implement Rate Limiting

Rate limiting caps how often an identity can hit an endpoint, protecting availability and
abuse-prone actions (credential stuffing, enumeration/scraping, DoS, cost blowups; OWASP API
Top 10 API4:2023). It **complements** authentication and authorization, it does not replace them.

## 0. Clarify before coding
- What are you protecting: login/abuse routes, expensive/metered endpoints, whole-API fairness?
- Do requests carry an authenticated session/account, or are the critical ones pre-login?
- Is there already an API gateway / dedicated rate-limit service and a shared store (Redis)?

## 1. Centralize enforcement (most important)
- Enforce rate limiting at ONE place: an **API gateway** or a **dedicated rate-limiting service**.
  One consistent policy, one enforcement point, one place to audit.
- Do NOT scatter home-grown counters inside every microservice (drift, duplicated bugs, gaps).
- **WAF caveat:** do not rely on the WAF as the main limiter. Its rules are coarse and it inspects
  every request, adding load and cost. Use the WAF/edge only to absorb crude volumetric floods;
  keep fine-grained, business-aware limiting at the central gateway/app tier.

## 2. Use a real algorithm (not a naive counter)
- Prefer **token bucket**: capacity = burst size, refill R/sec = long-run average; each request
  takes a token; empty bucket -> deny. Bursty yet bounded, and fair to idle clients.
- Avoid naive/fixed-window counters: boundary bursts (~2x at the edge) and no notion of fairness.
- Leaky bucket (steady output) and sliding window are fine alternatives; fixed-window counter is a
  cautionary baseline, not something to ship as-is.

## 3. Key on the right dimension
- Once authenticated, key by **session or account** (follows the identity across shared IPs, NAT,
  and IPv6 rotation). Use **IP only as the pre-auth fallback**, before a session exists.
- Multi-tier: pre-auth (IP + endpoint); post-auth (session/account + endpoint). Also per-API-key for
  client-app quotas, and tight per-endpoint caps on expensive/sensitive routes.

## 4. Make it correct at scale (distributed)
- With many instances, per-instance counters multiply the real limit (10 x 100 = 1000). Back the
  limiter with a **shared store (Redis) using atomic increments**; even a central gateway needs this.
- Consistency vs performance is a real trade-off; approximate algorithms trade a tiny error for speed.

## 5. Respond correctly
- On limit exceeded, return **HTTP 429 Too Many Requests** with a **Retry-After** header, and expose
  `RateLimit` / `X-RateLimit-*` (limit, remaining, reset) so good clients self-regulate.
- Clients should honor Retry-After and use **exponential backoff**. Decide the failure mode: fail safe.

## 6. CAPTCHA for session-less critical actions
- Critical pre-login endpoints (`/login`, `/signup`, `/password-reset`, `/otp`) have NO session and IP
  is weak, so rate limiting alone cannot tell a human from a bot. **Layer a CAPTCHA / human-challenge**
  on top, triggered on risk or after a few attempts (not every request). Keep the rate limit too.

## 7. Avoid the lockout DoS
- A hard account lockout after N failures lets an attacker lock out real users on purpose. Prefer
  progressive **throttling, backoff, and CAPTCHA** over hard permanent lockouts.

## Final checklist
- [ ] Enforced centrally (gateway / dedicated service), not per-microservice.
- [ ] Token bucket (or sliding/leaky), NOT a naive fixed-window counter.
- [ ] Keyed by session/account when authenticated; IP only pre-auth; per-endpoint tiers.
- [ ] Shared store (Redis) with atomic increments across all instances.
- [ ] 429 + Retry-After + RateLimit headers; clients use exponential backoff; fail safe.
- [ ] CAPTCHA layered on critical session-less endpoints.
- [ ] Throttle/backoff instead of hard lockouts; rate limiting sits alongside auth, not instead of it.
- [ ] WAF/edge used only for crude volumetric floods.

## Anti-patterns to refuse
- Per-microservice ad-hoc counters with no central policy.
- Relying on the WAF as the primary fine-grained limiter.
- Naive fixed-window counters shipped as-is.
- Keying only on IP when a session/account is available.
- Per-instance counters with no shared store (limit multiplies).
- Session-less login/reset with rate limiting but no CAPTCHA.
- Hard permanent account lockouts (self-inflicted DoS).
