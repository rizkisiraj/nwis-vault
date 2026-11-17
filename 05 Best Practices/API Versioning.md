---
tags: [practice]
---
# API Versioning

Up: [[Best Practices MOC]]

## The idea
A version in the API path (`/api/v1/...`) or header lets you change a contract without breaking existing clients: you ship `v2` beside `v1` and retire `v1` later. It matters most when clients update on a different schedule from the server, such as mobile apps.

## Why it matters
- Old clients keep working.
- Breaking changes become explicit decisions.

## In this codebase
Versioning here is a **prefix convention**, not a real multi-version practice: a grep of backend controllers finds `v1` only, no `v2`.
- **Backend prefixes by system:** controllers use `@RequestMapping("/api/v1/<system>")`, e.g. `/api/v1/nwis` (20 controllers), `/api/v1/ftsads` (5), `/api/v1/tgmdpweb/...`, `/api/v1/tgmdpvendor/...`, `/api/v1/healthCheck` (under `tgmdp_backend/.../controller/`). The system segment, not the version, is what separates the web console, vendor portal and mobile adapters; this is how one backend serves several frontends (see [[Integration Landscape]]).
- **Web console builds the URL from parts:** `tgmdp_web_frontend/src/shared/api/client.ts:46-50` creates axios with `apiVersion: 'v1'` and `basePath: 'tgmdpweb'`; the request interceptor (`:71-88`) calls `buildBaseURL(baseURL, apiVersion, basePath)`. A call can override: `authApi.ts:66-71` passes `basePath: ""` because `healthCheck` is not under `tgmdpweb`. A test (`additionalApis.endpoint.test.ts`) asserts the final URL, since "neither piece alone proves where a call lands".
- **Vendor portal hardcodes the full prefix:** `tgmdp_vendor_frontend/src/shared/constants.ts:22-31` uses `baseUrl: "/api/v1/tgmdpvendor/acceptance"`, so a version bump is a search-and-replace.
- **Unversioned legacy-shaped paths:** ESB inbound endpoints keep the legacy shape, e.g. `/ACPT_SERVICE/api` and `/MRM_ADS/api/WH` (backend controllers), because external systems call fixed URLs.
- **Legacy-compatible request shapes:** `NwisCheckVersionRequestDto` documents `GET /api/v1/nwis/checkVer` as the port of legacy `NWIS2_CheckVer_CheckVer`; the adapters keep legacy param names ([[Strangler Fig Migration and Legacy Parity]]).
- Related but different: app version checks and [[Over-the-Air Updates]] deal with client versions, not API versions.

## Check yourself
Which would be harder to change to `v2`: the web console or the vendor portal, and why?

## Related
- [[Strangler Fig]] · [[Feature Flags and Mock Toggles]] · [[Adapter]] · [[Web Console API Layer]] · [[Vendor Portal API Layer]] · [[Hybrid Shell Architecture Trade-offs]] · [[Best Practices MOC]]
