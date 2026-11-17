---
tags: [concept]
---

# Client-Server Architecture

A client asks, a server answers. The client (browser, mobile app) owns the user interface and sends requests over a network; the server owns the data and business rules and returns responses. Splitting the two lets many clients share one backend and lets each side change independently, at the cost of a network between them that can be slow or fail.

A useful corollary: never trust the client. Anything the client checks (a hidden button, a form rule) must be checked again on the server.

## In this codebase

One server, many different clients:

- **Server.** `tgmdp_backend` is a single Spring Boot app serving the admin web console, the vendor portal, the RN apps and the ESB. Controllers are grouped by caller, e.g. `controller/tgmdpweb/` (admin console), `controller/tgmdpvendor/` (vendor portal), `controller/nwisads/`, `controller/amsads/` (mobile modules).
- **Browser clients.** `tgmdp_vendor_frontend/src/lib/client.ts:44-48` creates an axios instance with `baseURL` from `VITE_API_BASE_URL`, a 60 s timeout and `withCredentials: true` (cookie auth). The admin console has the equivalent in `tgmdp_web_frontend/src/shared/api/client.ts`. Both are pure clients: they hold no business data of their own.
- **Embedded web client behind a shell.** `tgmdp-app-ams/src/services/api/client.ts:14` says "auth is handled at the RN bridge level for WebView flows": the React app runs in a WebView inside `tgmdp_rn_app`, and requests flow through `services/bridge/transport/bridge-request-executor.ts`, which executes an `IBridgeAPIRequest` natively. The native shell is the real network client; the web layer asks it. See [[Bridge API Request Flow and Auth]].
- **The server is also a client.** The backend and jobs call the ESB over SOAP (`EsbWsSecurityHandler`, `ESBPortType port = mdpEsbSoapService.getPort()` in `SyncTicketAllData.java:78`), and the ESB calls back into endpoints published by `CxfConfig`. So roles depend on which side starts the call. See [[What an ESB Is]].
- **Never trust the client, in practice.** The browser hides buttons by permission, but the backend repeats the check on each handler with `@PreAuthorize("hasAuthority('TASK_TGMDPx007:browse') and hasAuthority('SYS_預領料')")` (`PersonalMaterialsImportController.java:31`). See [[Authorization and RBAC]].
- **Legacy contrast.** In the legacy monolith, JSP pages were rendered on the server, so "the client" was just a browser showing HTML; see [[Server-rendered JSP vs React SPA]].

## Why it matters

Every bug report starts with "which side is wrong, client or server?" The network tab and the backend log are the two places to look.

## Related

- [[HTTP and REST]] · [[Stateless vs Stateful]] · [[Gateway]] · [[Layered Architecture]]
- [[Vendor Portal API Layer]] · [[Concepts MOC]] · [[System Overview]]
