---
tags: [concept]
---
# Cookies and CSRF

A cookie is a small value the server asks the browser to store and resend automatically with every request to that site. Sessions often ride on cookies, with flags like HttpOnly (hidden from JavaScript, limits XSS theft), Secure (HTTPS only) and SameSite (Strict/Lax/None: whether the cookie is sent on cross-site requests). Because the browser attaches cookies automatically, a malicious site can trigger a request as you: cross-site request forgery (CSRF). Defences: SameSite cookies, anti-CSRF tokens, and checking `Origin`/`Referer` headers.

The frontend side is usually just "send credentials with each request".

## In this codebase
- **The cookie.** After vendor OTP verification, `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/controller/tgmdpvendor/TgmdpvendorAuthController.java:318-323` builds the JWT cookie:
```java
ResponseCookie.from("access_token", response.getAccessToken())
        .httpOnly(true).secure(cookieSecure).path("/")
        .maxAge(Duration.ofHours(8))
        .sameSite(cookieSecure ? "None" : "Lax");
```
`cookieSecure` comes from the `COOKIE_SECURE` env var (`:60`): `Secure` + `SameSite=None` in deployed HTTPS, `Lax` for plain-HTTP local dev (browsers reject `None` without `Secure`).
- **Why `SameSite=None`.** The React SPAs are served from a different origin than the API, so cookies must travel cross-site. The frontends send them with `withCredentials: true` (`tgmdp_vendor_frontend/src/lib/client.ts:47`, same in `tgmdp_web_frontend/src/shared/api/client.ts:47`) and the backend answers CORS with an allow-list from `CORS_ORIGIN` (`config/CorsConfig.java:17`).
- **The price: CSRF.** `None` means any website can make the victim's browser send the cookie. Spring's CSRF filter is turned off (`security/WebSecurityConfig.java:60,79`: `.csrf(AbstractHttpConfigurer::disable)`) because a token scheme would need changes in every client. Instead `security/CsrfOriginCheckFilter.java:56-95` checks origin on every non-safe method: `Origin` header (fall back to `Referer`) must be in the allow-list, else `403 "Request origin not allowed"`. Requests with neither header pass, with the stated reason that native mobile clients have none and CSRF is a browser-cookie attack. It is registered before the auth filter (`WebSecurityConfig.java:168`), so forged requests are rejected before any token is read.
- **Legacy.** The JSP apps relied on the container's session cookie (`HttpSession` in `legacy/tgmdp_vendor_legacy/src/com/prj/login/Login.java:60`). I found no CSRF defence and no cookie flags configured in the legacy sources I searched (`web.xml`, Java, JSP); that is an absence in what I grepped, not proof it was never handled at the proxy.

## Why it matters
Misconfigured cookies cause both "randomly logged out" bugs (cookie not sent cross-site, `Secure` on HTTP) and real security issues.

## Check yourself
Why does the origin check let a request with no `Origin` and no `Referer` through, and why is that acceptable for the RN app but not a reason to drop the check for browsers?

## Related
- [[Authentication and Sessions]]
- [[Authorization and RBAC]]
- [[HTTP and REST]]
- [[Stateless vs Stateful]]
- [[tgmdp_backend Security and Auth]]
- [[Vendor Portal API Layer]]
- [[Vendor Portal Routing Auth and Permissions]]
- [[Web Console Routing Auth and Permissions]]
- [[FrontControllerFilter and ButtonBean Permissions]]
- [[Concepts MOC]]
- [[System Overview]]
