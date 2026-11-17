---
tags: [practice]
---
# Feature Flags and Mock Toggles

Up: [[Best Practices MOC]]

## The idea
A **feature flag** turns behaviour on or off through config without a deploy of new code. A **mock toggle** is a flag that swaps a real dependency (backend, ESB) for a fake one. Both help you ship unfinished work safely and develop when a dependency is unavailable. The cost is flag debt and untested combinations.

## Why it matters
- Decouples deploying from releasing.
- Lets front-end work continue before the backend exists.

## In this codebase
- **Web console mock switch per feature:** `tgmdp_web_frontend/src/features/document-download-area/api/index.ts:4-11` reads `VITE_USE_MOCK === "true"` and exports either `documentDownloadApiMock` or `documentDownloadApi`. The same line appears in several features (device inquiry, message push, key notification). `.env.example` documents `VITE_USE_MOCK` and `VITE_MOCK_DELAY_MS` ("simulate network"). `TaskRouteGuard.test.tsx:69,225` stubs the variable both ways.
- **Vendor portal:** `tgmdp_vendor_frontend/.env.example:13` declares `VITE_API_MOCK`.
- **Backend mock mode:** `SiteViewSyncService.java:40` reads `siteview.sync.mock` (default `false`); at `:96` it returns a fake success ("... (MOCK)") and saves a `MOCK-<uuid>` request instead of calling the ESB. Also `nwis.tc88.empty-template.enabled` (`NwisTemplateService.java:36`) and the dev-only `nwis.*.skip-enabled` flags (`CLAUDE.md:205`), which bypass the ESB.
- **RN shell:** `tgmdp_rn_app/configs/feature-flags.ts:25-40` turns module availability into booleans, e.g. `NWIS2: Boolean(moduleUrls.NWIS2)`: a module exists only if its URL is configured. It also holds WebView pool policy flags (`STRICT_3_LRU_ENABLED`, `AUTO_FALLBACK_ENABLED`, `ENABLE_RUNTIME_OVERRIDE`) with defaults in lines 18-44.
- **Fake ESB:** [[Why a Fake ESB Exists]] covers the system-level version: a whole stand-in service. [[Fake ESB Internals]] covers its request-header-driven error branches (not re-verified here).
- Risk shown by the repo's own warning: a skip flag left on makes a flow "look healthy" without hitting the ESB.

## Check yourself
What could go wrong if `VITE_USE_MOCK=true` is baked into a production build?

## Related
- [[Environment Separation]] · [[Gateway]] · [[Unit Testing]] · [[API Versioning]] · [[Mocking]] · [[Web Console API Layer]] · [[Best Practices MOC]]
