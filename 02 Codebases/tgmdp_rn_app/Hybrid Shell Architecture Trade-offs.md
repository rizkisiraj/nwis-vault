---
tags: [codebase, tgmdp-rn-app, concept]
---
# Hybrid Shell Architecture Trade-offs

> Why build one React Native shell that hosts React web apps in WebViews, instead of one big native app or a set of separate apps? This note lists the reasons visible in the code, then the costs the code pays for them, each with evidence. Every architecture is a trade; the useful skill is naming both sides.

Up: [[tgmdp_rn_app]] · [[Mobile Architecture Overview]] · [[System Overview]]

## The alternatives that were on the table

| Option | What it would look like |
|---|---|
| A. Keep the legacy Cordova apps | One install per business system, each with its own login and plugins |
| B. One big React Native app | Every business screen rewritten as RN components in one repo and one release train |
| C. **Native shell + web modules (chosen)** | One installed RN shell; each business module a Vite/React SPA loaded into a WebView and talking through a bridge |
| D. Pure web app in the phone browser | No install, but no camera/scanner/push/secure storage and no SSO with legacy apps |

## Why C was chosen (the benefits)

**1. Ship business fixes without an app-store release.** Modules are zip bundles with a version and a checksum, downloaded by `core/bundled-manager` (see [[WebView Pool and Bundle Loading]]). AMS CI publishes a new bundle on every push to `develop`. For an enterprise app distributed through enterprise signing, store and MDM release cycles are slow; web bundles are not. This is the strongest single reason.

**2. One login, one session, many systems.** The shell owns identity (`services/auth/`), and every module inherits it through `API_REQUEST`. Before, each Cordova app read a shared file and implemented its own checks. See [[Bridge API Request Flow and Auth]].

**3. Native power written once.** Camera, scanner, GPS, phone dial, push and biometrics live in the shell (`modules/tgmdp-*`). Eight guests share them through a typed protocol instead of eight copies of Cordova plugins.

**4. Incremental migration.** A module becomes "migrated" when its URL is configured (`configs/feature-flags.ts:22-40`); until then the menu launches the legacy app. Teams can move one business system at a time, which is the [[Strangler Fig]] pattern applied to a phone.

**5. Web skills and web tooling for business screens.** AMS and NWIS2 are ordinary Vite + React + Zustand + Vitest projects. They can be developed in a browser-like loop (URL mode against a dev server) and staffed by web developers. The same team also writes the browser consoles ([[tgmdp_web_frontend]], [[tgmdp_vendor_frontend]]).

**6. Faithful legacy look.** The legacy apps were HTML/CSS in Cordova. Rebuilding them as HTML/CSS in a WebView makes pixel parity with legacy (a hard requirement here) much cheaper than re-creating the look in native components. AMS literally ships `style_ams2S.css`.

## What it costs (with evidence)

**1. A protocol to design, version and defend.** The bridge has some 30 message types, two message styles (explicit `NATIVE_REQUEST` + `operation` and older bare types), a boot handshake, request IDs, document IDs and boot IDs (see [[Bridge Engine and Message Protocol]]). The guest has its own copy of these types (`tgmdp-app-ams/src/utils/bridge/types.ts`) with no shared package, so the two sides drift unless someone checks. AMS comments cite shell merge requests by number (`tgmdp-app-ams/src/services/api/ams.ts:6`), which is what coordinating two release trains looks like in practice.

**2. Errors lose fidelity at every hop.** An HTTP error becomes an axios error, then a handler error, then a `{status:false, message, errorData}` reply, then a rejected promise in the guest. AMS needs a 40-line helper just to recover the backend's real message (`src/utils/bridge/api.ts:67-113`). The team has hit this before: a stored note records a period when backend error text did not reach the NWIS2 UI at all.

**3. Retries can repeat non-idempotent calls.** The `api` retry policy retries on 500/502/503/504 with no check on HTTP method (`services/bridge/resilience/bridge-retry-policy.ts:45-56`). On top of that, if the routed handler fails, `modules/bridge-engine/handler/inbound.ts:46-53` re-runs the same message through the legacy path, which calls `APIRequest` again (`inbound-legacy-helpers.ts:179-182`). A `POST` that reached the server but returned 500 may therefore be sent more than once. I could not find a guard against this; treat it as an open question to verify before relying on exactly-once writes. See [[Database Transactions]] and idempotency.

**4. The guest is less isolated than "the module never sees the token" suggests.** The shell attaches the token to HTTP calls, but `REQUEST_SESSION` hands `authToken` to the guest (`services/bridge/session/bridge-session-builders.ts:174-185`) and the Cordova shim can read USERFILE. The vendor shell made the stricter choice and never sends the token to the web layer ([[tgmdp_vendor_rn|Vendor Shell Bridge and API Proxy]]). Compare and decide which you would copy.

**5. Memory and startup.** Each WebView is a browser engine. The shell needs a pooled, LRU-evicted, memory-pressure-aware WebView runtime with guardrails and a benchmark just to make module switching feel native. Option B would not need any of it.

**6. Two of everything in testing.** Bridge code is excluded from the AMS coverage gate because "it requires mobile app runtime, untestable in jsdom" (`tgmdp-app-ams/vite.config.ts:194-196`). The seam between the two halves, the most failure-prone part, is the part neither repo's unit tests cover end to end. Real confidence needs device runs (the workspace has a `ui-test-runner` workflow driving simulators for that).

**7. Two lifecycles inside one app.** A cached WebView keeps its JS alive while the shell navigates elsewhere. AMS has special code for "re-entered after exit" (`src/hooks/useWebViewBoot.ts:50-55`), for an exit that must log out before the shell takes over (`src/stores/userStore.ts:11-17`), and for host back-button events (`App.tsx:98-123`). A browser tab never has to think about these.

**8. Security surface.** `allowFileAccessFromFileURLs` is on for `file://` bundles, downloaded code runs with a bridge to identity and hardware, and every bridge input must be validated (size limits, URL patterns, navigation gate, origin check for Cordova calls). The shell does all of this, but it is work Option B would mostly not need. See [[Defensive Programming]].

## A balanced verdict

For this organisation (many legacy systems, a mandated parity with old screens, a long migration, enterprise distribution and a web-heavy team) Option C buys deployment speed and gradual migration at the price of a protocol, a runtime and a testing gap. The code shows a team that understands the costs: hardening profiles, typed boot states, a pool benchmark, a refusal to enable unsigned OTA. The two items most worth a reader's attention are the duplicated-write risk (cost 3) and the token exposure (cost 4).

## Questions to check understanding

1. Which benefit disappears if the enterprise later gets same-day app-store releases? Does the architecture still pay for itself?
2. Propose a change that removes cost 3 without losing the safe-rollout fallback.
3. Would you make the bridge types a shared npm package used by both the shell and the guests? What new problem does that create?

## Related

- [[tgmdp_rn_app]] · [[Mobile Architecture Overview]] · [[System Overview]]
- [[Bridge Engine and Message Protocol]] · [[Bridge API Request Flow and Auth]] · [[WebView Pool and Bundle Loading]] · [[Native Modules and Legacy Coexistence]] · [[tgmdp_vendor_rn|Vendor Shell vs TGMDP Shell]] · [[tgmdp-app-ams]]
- Concepts: [[Hybrid Mobile Apps]] · [[WebView and Native Bridge]] · [[Single Page Applications]] · [[Monorepo vs Polyrepo]] · [[Testing Pyramid]] · [[Error Handling]] · [[Database Transactions]]
- Patterns: [[Strangler Fig]] · [[Gateway]] · [[Anti-Corruption Layer]]
- Practices: [[Defensive Programming]] · [[API Versioning]] · [[Feature Flags and Mock Toggles]] · [[Test Coverage Gates]]
- New: [[Idempotency]] · [[Over-the-Air Updates]]
