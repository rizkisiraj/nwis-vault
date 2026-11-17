---
tags: [concept]
---

# Stateless vs Stateful

A stateful server remembers things between requests (an in-memory session, a wizard step). A stateless server treats each request as complete: everything needed arrives with it, often as a token. Stateless services scale and recover easily because any instance can serve any request; stateful ones need sticky routing or shared session storage.

State does not vanish; it moves to the client, a token, or a database.

## In this codebase

- **Legacy: stateful by session.** `legacy/TGMDP/src/com/prj/login/Login.java:63-64` stores `UserBean` and `ButtonBean` (the user's permitted buttons) in the servlet `HttpSession`; `com/sys/filter/FrontControllerFilter.java:24,64` reads them back on every action. Restart the app server and every user is logged out; run two servers and you need sticky routing. See [[Session ButtonBean vs JWT Authorities]] and [[FrontControllerFilter and ButtonBean Permissions]].
- **Modern backend: stateless by token.** `WebSecurityConfig.java:84-85` sets `SessionCreationPolicy.STATELESS`. `AuthTokenFilter.java:220-232` finds the credential on each request, first in an `access_token` cookie, otherwise in an `Authorization: Bearer ...` header. Any backend instance can serve any request. See [[tgmdp_backend Security and Auth]] and [[Authentication and Sessions]].
- **Where the state went.** Into the token (JWT, authorities), into the browser cookie, and into DB2. The vendor client sends the cookie with `withCredentials: true` (`tgmdp_vendor_frontend/src/lib/client.ts:47`) and keeps the username in a persisted zustand store (`src/features/auth/store/index.ts:16-32`, key `auth-storage`).
- **Hidden state that is still there.** A "stateless" backend still caches in memory: `MediatorService` holds the parameter table in two `HashMap`s per JVM and reloads when a timestamp changes (`MediatorService.java:50-72`). A second instance would hold its own copy and refresh independently. Quartz in `tgmdp_jobs` is configured `isClustered: false` (`application.yml:89`), so scheduler state is single-instance.
- **Mobile.** In the RN shell the session lives natively; the web layer in the WebView does not hold the token, because "auth is handled at the RN bridge level" (`tgmdp-app-ams/src/services/api/client.ts:14`). See [[Bridge API Request Flow and Auth]] and [[AMS Entry and Login Flow]].
- **CSRF note.** The backend disables CSRF (`WebSecurityConfig.java:79`) yet accepts cookie auth, and compensates with `CsrfOriginCheckFilter` (added before the token filter, line 168). See [[Cookies and CSRF]].

## Why it matters

It tells you what breaks when you add a second server or restart one: legacy sessions die, JWTs survive, in-JVM caches diverge.

## Check yourself

The backend is `STATELESS`, yet `MediatorService` keeps data in fields. Which of the two properties (stateless / horizontally scalable) does that put at risk, and how does the timestamp check mitigate it?

## Related

- [[Authentication and Sessions]] · [[HTTP and REST]] · [[Client-Server Architecture]] · [[Store Pattern]]
- [[WebWork Actions vs Spring Controllers]] · [[Vendor Portal Routing Auth and Permissions]] · [[Concepts MOC]] · [[System Overview]]
