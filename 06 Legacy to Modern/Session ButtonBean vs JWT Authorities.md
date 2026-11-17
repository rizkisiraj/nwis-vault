---
tags: [legacy]
---

# Session ButtonBean vs JWT Authorities

> Up: [[Legacy to Modern MOC]]

How the system decides who may press which button, before and after.

## Before: ButtonBean in the session

A permission is a string `<TASK_ID>_<BUTTON_ID>`, for example `TGMDPa002_browse`. At login, one query flattens the user's roles into a `ButtonBean` stored in the HTTP session. A servlet filter then checks the string on every `*.action` request. Failure calls `session.invalidate()`, so the user is logged out, not shown a 403. See [[FrontControllerFilter and ButtonBean Permissions]].

## After: authorities in a JWT

The backend is stateless (`SessionCreationPolicy.STATELESS`). Each request carries a signed JWT, and the user's `GrantedAuthority` list holds `TASK_TGMDPs011:browse`-style strings (one-to-one with the ButtonBean model) plus `SYS_*` system authorities. Controllers guard with `@PreAuthorize("hasAuthority('TASK_TGMDPs011:update')")`. See [[tgmdp_backend Security and Auth]].

## Side by side

| Aspect | Legacy | Modern |
|---|---|---|
| State lives in | Server HTTP session | Signed JWT (plus `a_login_jwt` table for revocation) |
| Permission key | `TGMDPa002_browse` | `TASK_TGMDPa002:browse` |
| Check | Servlet filter, `buttonList.contains` | `@PreAuthorize` on controllers |
| System licence | `ButtonBean.checkSysAuth` | `SysAuthRules` port, `SYS_*` authorities |
| Denied | Session invalidated, back to login | 401 JSON via `AuthEntryPointJwt` |

## What was kept

The task code and button names are kept on purpose, so reviews can compare both sides. The web console mirrors them too, with `hasSysAuth` ported from `checkSysAuth` and a guard that logs out like legacy. See [[Web Console Routing Auth and Permissions]].

## Related

- [[Authorization and RBAC]] · [[Authentication and Sessions]] · [[Stateless vs Stateful]]
- [[Legacy to Modern MOC]] · [[DBDAO vs Spring Data Repositories]]
