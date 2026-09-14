---
name: implement-webview-jsbridge-security
description: Secure mobile WebViews and the JavaScript-to-native bridge. A WebView runs web content inside your app, and a bridge (Android addJavascriptInterface / @JavascriptInterface, iOS WKScriptMessageHandler) lets that web content call native code. The core danger: ANY script running in the WebView (an ad, an iframe, or an XSS-injected script) can call the bridge, and the bridge cannot reliably verify which frame/origin called it (do not trust WebView.getUrl()). So control BOTH ends: load only your own trusted content over HTTPS with a URL allowlist (never third-party/user content in a bridged WebView; use a separate bridge-less WebView for untrusted content), and keep the bridge minimal (no tokens, no file access, no sensitive actions), treating every call as untrusted input to validate and authorize. On iOS use WKWebView message handlers and check the frame origin; disable unneeded features (file access, JS if unused) and enforce a Content Security Policy. Use for any hybrid app or app embedding web content.
---

# Implement WebView & JS-Bridge Security

A WebView embeds web content in your app; a JS-to-native bridge lets that content call native code
(read a token, open a file, trigger an action). The risk: every script in the WebView can call the
bridge, and you cannot reliably tell which one, so untrusted content becomes native power.

## 0. Clarify before coding
- What does the WebView load: only your own pages, or any third-party/user content?
- Does the bridge expose anything sensitive (tokens, files, actions)? Does it need to?
- Android, iOS, or both (hybrid)?

## 1. Understand the two risks
- **What loads:** untrusted or third-party content (ads, iframes, external pages, user HTML, or an XSS
  bug) means someone else's script runs inside your app.
- **The bridge:** it exposes native capabilities to JavaScript. Any script in the WebView can call it,
  and there is no reliable way to verify the calling frame's origin (WebView async; do not trust
  `WebView.getUrl()`).

## 2. Control what loads (in-bound)
- Load ONLY your own trusted content over **HTTPS**; **allowlist** the exact URLs the WebView may open.
- **Never** load third-party or user-supplied content in a WebView that has a bridge.
- If you must render untrusted content, use a **separate WebView with no bridge** (and no sensitive context).
- Enforce a **Content Security Policy** to limit which scripts can run, and reduce XSS risk in your own pages.

## 3. Control the bridge (out-bound)
- Keep the bridge **minimal**: do not expose session tokens, file access, or sensitive actions through it.
  The smaller the surface, the smaller the blast radius.
- Treat **every bridge call as untrusted input**: validate parameters and authorize the action, exactly
  like a network request. Do not assume the caller is your page.
- Android: prefer safer messaging (`WebViewCompat.postWebMessage` / `WebMessageListener` with an
  allowlisted origin) over a broad `@JavascriptInterface`; expose only trusted, per-origin methods.
- iOS: use **WKWebView** `WKScriptMessageHandler`; check `message.frameInfo` (origin / isMainFrame);
  keep the set of handlers small.

## 4. Disable what you do not need
- Turn off JavaScript if the content does not need it; disable file access
  (`setAllowFileAccess(false)`, `allowFileAccessFromFileURLs`/`allowUniversalAccessFromFileURLs` off);
  do not mix trusted and untrusted content in one WebView.

## Final checklist
- [ ] WebView loads only trusted, allowlisted HTTPS content; no third-party/user content with a bridge.
- [ ] Untrusted content (if any) runs in a separate, bridge-less WebView.
- [ ] Bridge is minimal; exposes no tokens/files/sensitive actions.
- [ ] Every bridge call validated and authorized as untrusted input.
- [ ] iOS: WKWebView message handlers with frame-origin checks; small handler set.
- [ ] Unneeded features disabled (file access, JS if unused); CSP enforced.
- [ ] No reliance on WebView.getUrl() for security decisions.

## Anti-patterns to refuse
- Loading third-party/user content in a WebView that has a native bridge.
- Exposing tokens, file access, or sensitive actions through `addJavascriptInterface`/message handlers.
- Trusting the calling frame's origin via `WebView.getUrl()`.
- A broad, catch-all bridge instead of a minimal, validated, per-origin one.
- Leaving file access / universal access enabled when not needed.
