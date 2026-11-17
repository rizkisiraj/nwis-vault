---
tags: [codebase, tgmdp-app-ams]
---
# tgmdp-app-ams (repo map)

> The **AMS web layer**: the 基站障礙 (base-station fault handling) app that field engineers use to see, accept, reply to and photograph fault tickets. It is a Vite + React 19 single page app with **no native code**. It runs inside a WebView in the [[tgmdp_rn_app]] shell and does every backend call, GPS fix and camera shot by sending a message to that shell.

Up: [[System Overview]] · Overview: [[Mobile Architecture Overview]] · Host: [[tgmdp_rn_app]] · Sibling guest: [[tgmdp-app-ams|tgmdp-app-nwis2 Sibling Micro-App]]

## What problem this repo solves

AMS2 was a Cordova/jQuery app (`ams2-app-legacy/` in the workspace). This repo rebuilds it screen by screen in React, with **legacy parity** as the rule: the same menus, the same dialog texts, the same validation, even the same CSS (`src/style_ams2S.css`). Comments cite the legacy source they copy, for example `// Legacy AdapterUtility.js:69-110` in `src/hooks/useSessionLost.ts:19`. When you read a strange-looking rule, look for the legacy citation next to it before "fixing" it.

It is one of the guest modules registered in the shell as `AMS2` (`tgmdp_rn_app/constants/modules.ts:24-29`). The package's own name is `基站障礙` with `appId: "2"` (`package.json:2-3`).

## Tech stack at a glance

| Layer | Tool | Where |
|---|---|---|
| UI | React 19 | `package.json` |
| Build | Vite 7, `base: './'` for `file://` loading | `vite.config.ts:141` |
| Language | TypeScript 5.9, `strictTypeChecked` ESLint preset | `eslint.config.js:16` |
| Routing | react-router-dom 7 with **HashRouter** | `src/main.tsx:28` |
| State | Zustand 5 | `src/stores/` |
| Styling | Tailwind CSS 4 plus the legacy stylesheet | `src/index.css`, `src/style_ams2S.css` |
| Maps / charts | `@react-google-maps/api`, Chart.js 3 | site map and statistics pages |
| Tests | Vitest 4 + Testing Library + jsdom, **100% thresholds** | `vite.config.ts:153-249` |
| Packaging | `npm run build-zip` via `@tgmdp/build-utils` → `.bundle/<version>.zip` | `scripts/build.js` |
| Spec workflow | OpenSpec (`openspec/specs`, `openspec/changes`) | `README.md` |

## Top-level layout

```
src/
├── main.tsx              createRoot + HashRouter + error boundary
├── App.tsx               boot hooks, global dialogs, lazy routes
├── pages/                one folder per legacy menu (a001m1, d001, C001M1, b001M2, e001, f001,
│                         ticketShare, DataShare, HIWshare, IronTower, gnb)
├── hooks/                one folder per menu + root hooks (useAppLogin, useSessionLost...)
├── components/           shared UI (common, layout, menu, ticket, Form, inad, mplus...)
├── services/
│   ├── api/              ams.ts, sd1.ts, cwa.ts, bridge.ts (bridgeClient), client.ts (direct axios,
│                         used only as a browser fallback for the public checkVer call, ams.ts:652-658)
│   ├── bundleUpdate.ts   OTA version check through the shell
│   └── locationService.ts, mplusService.ts, ironTowerService.ts
├── stores/               Zustand stores (user, webView, loading, ticket context, ...)
├── utils/bridge/         the guest half of the bridge: core, api, boot, navigation, camera, push...
├── types/                request/response DTO types per domain
├── constants/            entry and GPS messages (legacy texts)
└── test/                 setup.ts, mockBridge.ts, mockAxios.ts, fixtures
```

About 287 source files and 172 test files.

## Backend endpoints used

All calls go through the shell. The paths the app sends (grep of `src/services/api/*.ts`):

| Prefix | Use | Backend package |
|---|---|---|
| `/api/v1/ams/*` | AMS-specific: `checkVer`, `checkAuthority`, logout, tickets, push, photos, gNB tests | AMS adapters in [[tgmdp_backend]] |
| `/api/v1/sd1/*` | Site data shared with other systems: antenna, one-week data, SMR, technology | SD1 adapters |
| `/api/v1/tgmdp/auth/appLogin` | Shared app-entry login used by every guest module | TGMDP auth |
| `https://opendata.cwa.gov.tw/api/v1/rest/datastore/...` | Weather for HIW (high-altitude work), an **external** public API | not our backend |

Backend endpoints are sent as relative paths; the shell adds the base URL and the token. See [[Bridge API Request Flow and Auth]].

The weather calls (`src/services/api/cwa.ts:18,27`) are worth a second look. They are absolute URLs to a third-party service, which the shell's `api` hardening profile allows (`^https?://` in `bridge-hardening-profiles.ts:22`), and the open-data authorisation key is written into the source. Even a free public key in a client bundle is readable by anyone who unzips the bundle; see [[Secrets Management]].

## Notes in this folder

- [[AMS Features and Routing]]: the menus, the pages and how routing works in a WebView.
- [[AMS Bridge Client]]: the guest half of the bridge: sending messages, pending-request registries, error mapping, session-lost handling.
- [[AMS Entry and Login Flow]]: what happens between "WebView loaded" and "menu shown", with a sequence diagram.
- [[AMS State Hooks and Testing|AMS State and Hooks]]: Zustand stores and the hook-per-API-call convention.
- [[AMS State Hooks and Testing|AMS Testing and the Coverage Gate]]: Vitest setup, the bridge mock, and what the 100% gate really measures.
- [[tgmdp-app-ams|tgmdp-app-nwis2 Sibling Micro-App]]: the NWIS2 guest built on the same template.

## How to run it

```bash
npm install            # needs internal GitLab access for shared-module and @tgmdp/build-utils
npm run dev            # Vite dev server; point the shell's MODULE_URLS.AMS2 at it in URL mode
npm run test:coverage  # Vitest with the 100% thresholds
npm run build-zip      # the OTA bundle the shell downloads
```

In a plain browser the app shows the menu shell but every API call rejects with "Native bridge not available" (`src/services/api/bridge.ts:22-24`), because there is no `window.ReactNativeWebView`. Real work needs the shell.

CI (`.gitlab-ci.yml`) builds the zip on `develop` and uploads it to the module registry as `AMS2`.

## Try it yourself

- Open `src/App.tsx` and count the routes. Then open `src/components/menu/MenuList.tsx` and map each menu tile to a route.
- Pick one hook in `src/hooks/ticket/` and follow it to `AMS_API` in `src/services/api/ams.ts`, then to `bridgeClient`, then to `sendApiRequest`. That chain is the whole data path on the guest side.

## Related

- [[System Overview]] · [[Mobile Architecture Overview]] · [[tgmdp_rn_app]] · [[tgmdp_backend]]
- [[AMS Features and Routing]] · [[AMS Bridge Client]] · [[AMS Entry and Login Flow]] · [[AMS State Hooks and Testing|AMS State and Hooks]] · [[AMS State Hooks and Testing|AMS Testing and the Coverage Gate]] · [[tgmdp-app-ams|tgmdp-app-nwis2 Sibling Micro-App]]
- Concepts: [[Single Page Applications]] · [[Hybrid Mobile Apps]] · [[WebView and Native Bridge]] · [[Client State Management]] · [[Component-Based UI]] · [[Routing]] · [[Type Systems]]
- Patterns: [[Custom Hooks]] · [[Store Pattern]] · [[Feature-Sliced Structure]] · [[Strangler Fig]] · [[Facade]]
- Practices: [[Test Coverage Gates]] · [[Unit Testing]] · [[Linting and Formatting]] · [[Clean Code Naming]]
- New: [[Strangler Fig Migration and Legacy Parity|Legacy Parity]] · Spec-Driven Development
