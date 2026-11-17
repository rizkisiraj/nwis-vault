---
tags: [concept]
---
# Authentication and Sessions

Authentication proves who you are (password, OTP, captcha, token). After success the server remembers you through a session: either a server-side record keyed by an id sent back in a cookie (stateful), or a signed token the client presents on each call (stateless, e.g. JWT). Logout or expiry invalidates it.

Authentication answers "who", not "what may you do"; that is authorization. A pure JWT cannot be revoked before it expires unless the server also keeps a record of issued tokens.

## In this codebase
Legacy and modern use opposite session models.
- **Legacy: server-side `HttpSession`.** `legacy/tgmdp_vendor_legacy/src/com/prj/login/Login.java:60-62` stores the logged-in user object and a permission object in the session: `session.setAttribute("UserBean", userbean); session.setAttribute("ButtonBean", buttonbean);`. The browser only holds the container's session id; the state sits in server memory (see [[FrontControllerFilter and ButtonBean Permissions]]).
- **Modern: stateless JWT.** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/security/WebSecurityConfig.java:85` sets `SessionCreationPolicy.STATELESS`, so Spring never creates an `HttpSession`. Tokens are HS256-signed in `utils/JwtUtility.java:43,104` (`Keys.hmacShaKeyFor(...)`, `.signWith(key, SignatureAlgorithm.HS256)`). Why: any backend instance can serve any request without shared session memory.
- **Made revocable.** `security/AuthTokenFilter.java:62-70` first checks signature/expiry, then `jwtService.validateTokenWithDatabaseJWT(jwt)`, which queries `a_login_jwt ... WHERE jwt = :jwt AND STATUS = '0'` (`repository/ALoginJwtRepository.java:51`). Logout flips the status. Cost: one DB query per request. The filter then has per-client branches: it skips the DB-token rule for vendor and registry URLs (`AuthTokenFilter.java:98-112`, comparing `requestUrl.contains("/tgmdpvendor/")`), because vendor tokens are never stored in `a_login_jwt`, so a vendor logout cannot revoke early.
- **Vendor login is multi-step.** Captcha -> OTP send -> OTP verify (`service/tgmdpvendor/TgmdpvendorAuthService.java`): captcha single-use via `markCaptchaAsUsed` (`:281`), OTP valid `OTP_EXPIRY_MINUTES = 5`, lock after `MAX_RETRY_ATTEMPTS = 3` (`:44-45`), OTP delivered by inserting a row into `A_MPLUSVEN_LOG` (`:365`). Captcha and OTP state live in DB tables, not in memory, so the same "no server memory" idea applies.
- **Web console SSO.** `controller/tgmdpweb/TgmdpWebAuthController.java:51` reads an `iv-user` request header (set by a fronting proxy) as identity. Only safe if nothing else can reach the backend directly.
- **Frontend side.** `tgmdp_vendor_frontend/src/lib/client.ts:133-139` turns a 401 (outside the auth endpoints) into automatic logout/redirect.

## Why it matters
Every protected screen depends on knowing the session is valid and what to do when it expires. The hybrid stateless-plus-DB design explains odd behaviour: a "valid" JWT can still be rejected.

## Check yourself
A vendor's cookie is stolen. They click logout. Is the stolen token still accepted? Which line shows why?

## Related
- [[Authorization and RBAC]]
- [[Cookies and CSRF]]
- [[Stateless vs Stateful]]
- [[Secrets Management]]
- [[tgmdp_backend Security and Auth]]
- [[Vendor Portal Routing Auth and Permissions]]
- [[Vendor Portal API Layer]]
- [[Vendor Portal State and Forms]]
- [[tgmdp_vendor_frontend]]
- [[Bridge API Request Flow and Auth]]
- [[Concepts MOC]]
- [[System Overview]]
