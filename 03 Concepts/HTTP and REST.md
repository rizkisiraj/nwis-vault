---
tags: [concept]
---

# HTTP and REST

HTTP is the request/response protocol of the web: a request has a method (GET, POST, PUT, DELETE), a URL, headers and an optional body; the response has a status code (2xx ok, 4xx client error, 5xx server error), headers and a body. REST is a style on top of it: URLs name resources, methods say what to do with them, and each request is self-contained.

In practice many enterprise APIs are "REST-ish", for example everything is a POST returning JSON. The status code and body convention matters more than purity.

## In this codebase

- **Modern backend: resource URLs plus an envelope.** `PersonalMaterialsImportController.java:24-27` is `@RestController @RequestMapping("/api/v1/tgmdpweb/pre-requisition/personal-materials")`, with `GET /init`, `GET` (search), `GET /export`, `POST /import` (multipart). Every handler returns `ResponseEntity<ApiResponse<T>>` and copies the envelope's code into the HTTP status: `ResponseEntity.status(response.getCode()).body(response)`. `ApiResponse<T>` is `dto/ApiResponse.java`. The `/api/v1/<app>/...` prefix is also the versioning scheme: [[API Versioning]].
- **A deliberate REST limit: only GET and POST.** The vendor client types its axios wrapper to expose just those two (`tgmdp_vendor_frontend/src/lib/client.ts:32-39`, comment "the WAF blocks PUT/PATCH/DELETE"). A corporate firewall rule is therefore enforced by the compiler. So "update" and "delete" travel as POST. This is the REST-ish compromise in action.
- **Status-code translation on the client.** The same file maps axios errors (`Network Error`, `timeout`, `Request failed with status code 4xx/5xx`) to Chinese messages (`translateAxiosMessage`, lines 168-197).
- **Legacy: action names, not resources.** `legacy/TGMDP/src/xwork.xml:43-45` maps `<action name="Home" class="com.prj.login.Home" method="execute">` to a JSP result. URLs are verbs (`Login`, `Logout`, `begin`, `update`), pages are server-rendered, and state rides in the session. See [[WebWork Actions vs Spring Controllers]].
- **SOAP and 200-with-failure.** ESB traffic is HTTP too but POST-only XML; the reply header carries a status (`S` / `F`) regardless of HTTP 200. See [[SOAP vs REST]] and `MdpSoapEndpoint`. The fake ESB's REST side returns `400` plus status `20002` for bad JSON (`tgmdp_fake_esb/.../validation/B1ValidationHandler.java:20-41`).

## Why it matters

Reading the network tab is the fastest way to understand what any screen really does; the URL and status you see there map straight to a controller method.

## Check yourself

Why would an "update" in this system be a `POST` rather than a `PUT`, and where in the code is that decision enforced?

## Related

- [[Client-Server Architecture]] · [[Serialization and DTOs]] · [[Error Handling]]
- [[Vendor Portal API Layer]] · [[Vendor Portal Feature Trace - TGMDPVENc003 Serial Number Query]] · [[What an ESB Is]]
- [[Concepts MOC]] · [[System Overview]]
