---
tags: [concept]
---
# Authorization and RBAC

Authorization decides what an authenticated user may do. Role-based access control (RBAC) assigns permissions to roles and roles to users, so you manage "admin" or "vendor clerk" instead of individuals. Checks must happen on the server; hiding a button in the UI is only a convenience. Finer grain appears as per-function or per-button permissions.

## In this codebase
The model is inherited from legacy: **roles map to (task, button) pairs, plus a "system" category**.
- **Legacy.** `legacy/TGMDP_legacy_new/src/com/sys/presentation/ButtonBean.java:161-165` `checkButton(button)` builds `task_id + "_" + button` and tests it against the user's list. JSPs call it to render or hide buttons (`legacy/tgmdp_vendor_legacy/TGMDP_VENDOR/TGMDPVENc001/TGMDPVENc001Q1.jsp:82`, `button.checkButton("browse")`), and `checkSysAuth` (`:172`) tests system-level access. `Login.java:58-62` loads it into the session at login.
- **Modern: permissions become authority strings.** `service/JwtService.java:160-181` loops over role->task->button rows and adds `TASK_<taskId>:begin` and `TASK_<taskId>:<buttonId>`; `:131-135` adds one `SYS_<sys>` per system grant. Same data, new carrier (Spring `GrantedAuthority`).
- **Declarative check on each endpoint.** `controller/tgmdpweb/projectAcceptance/PoCategoryController.java:48-68`: `@PreAuthorize("hasAuthority('TASK_TGMDPs011:browse') and hasAuthority('SYS_工程驗收')")`. Method security is switched on at `security/WebSecurityConfig.java:18` (`@EnableMethodSecurity`), about 150 files use `@PreAuthorize`. This is the server-side check the UI buttons only mirror. URL-level rules in `WebSecurityConfig` stay coarse (`permitAll` vs `authenticated()`).
- **A ported quirk.** `security/SysAuthRules.java:14-34` copies legacy `checkSysAuth`: some systems accept a superset (`"鐵塔載重" -> 基站巡檢 or 基站障礙`), and the default is a **substring** match because stored names are sometimes longer than codes. Bug-for-bug parity: it keeps behaviour identical for users who already have these grants.
- **Failures go through the shared handler.** `@PreAuthorize` denial raises `AuthorizationDeniedException`, mapped to 403 in `config/GlobalExceptionHandler.java` (see [[Error Handling]]).

## Why it matters
A missing server-side check is a security hole even if the UI looks locked down. Here one `@PreAuthorize` per endpoint is the whole defence for staff features, so a new endpoint without one is open to any logged-in user.

## Check yourself
`TASK_TGMDPs011:browse` and `SYS_工程驗收` are both required. Which of the two came from per-button data, and which from the system category?

## Related
- [[Authentication and Sessions]]
- [[Cookies and CSRF]]
- [[Interceptor and Filter Chain]]
- [[Defensive Programming]]
- [[tgmdp_backend Security and Auth]]
- [[FrontControllerFilter and ButtonBean Permissions]]
- [[Vendor Portal Routing Auth and Permissions]]
- [[Vendor Portal Feature Trace - TGMDPVENc003 Serial Number Query]]
- [[AMS Entry and Login Flow]]
- [[Concepts MOC]]
- [[System Overview]]
