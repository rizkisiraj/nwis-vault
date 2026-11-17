---
tags: [concept]
---
# Hybrid Mobile Apps

Up: [[Concepts MOC]]

## The idea

A hybrid app mixes native code with web content. A native shell provides the app icon, login, permissions and device features, while screens are web pages shown inside a WebView. Cordova and React Native shells with web modules are examples.

The trade is speed of delivery against polish: web modules can be updated without an app store release, but they depend on a bridge to reach the camera, storage or secure keychain.

Key points:

- **Shell**: owns startup, navigation and native capabilities.
- **Web modules**: independently built SPAs loaded into WebViews.
- **Bridge**: a message protocol between JavaScript and native code.
- **Updates**: web bundles can ship over the air.

## In this codebase

Both generations are hybrid; what changed is who owns what.

- **Legacy (Cordova)**: `ams2-app-legacy/AMS2/` is a Cordova project (`config.xml`, `package.json` lists `cordova-hot-code-push-plugin`, `cordova-plugin-inappbrowser`, `-dialogs`, `-spinner-dialog`). The whole app is HTML, jQuery and a global `_APP_DATA` inside the Android/iOS WebView; the native side is just plugins. See [[Legacy Cordova App Anatomy]].
- **Modern shell with many modules**: `tgmdp_rn_app` (Expo / React Native). `configs/app-config.*.json` lists `MODULE_URLS` (AMS2, BTS, NWIS2, ACPT, ... 17 names) and a `WEBVIEW_POOL_POLICY` (`strict-3-lru`). `app/[module].tsx` redirects to `/webview/<module>`. A seed bundle `assets/modules/AMS/0.0.0.zip` ships in the app. See [[RN Shell App Structure]], [[WebView Pool and Bundle Loading]].
- **Modern shell with one module**: `tgmdp_vendor_rn/app/index.tsx` re-exports `core/webview/infrastructure/WebViewScreen`; `MODULE_ID` is `ACPTFirms`. The shell does the version gate, permissions, token proxy and native camera/scanner. See [[tgmdp_vendor_rn]].
- **The web layer**: [[tgmdp-app-ams]] is a React SPA that talks to native only through `window.ReactNativeWebView.postMessage` (`src/utils/bridge/core.ts:67-96`). It also contains a shim path for legacy behaviours: the shell injects a Cordova-compat shim (`tgmdp_rn_app/core/communications/cordovaShim.template.ts`, "if (window.cordova) return") so old-style calls can still work. See [[Native Modules and Legacy Coexistence]].

## Legacy vs modern

Cordova shipped HTML locally and replaced it with hot-code-push (`chcp.json`, `chcp.manifest`). The RN shell is a real native app with its own screens and secure storage, and the web is one component inside it. Web and shell are versioned and released separately.

## Why it matters

Most mobile work here is web code, so the shell's rules, not the app store, shape how it is built and released. Costs: a bridge to maintain and two runtimes to debug ([[Hybrid Shell Architecture Trade-offs]], [[Mobile Architecture Overview]]).

## Related

- [[WebView and Native Bridge]] · [[Single Page Applications]] · [[Monorepo vs Polyrepo]] · [[Caching]] · [[Strangler Fig]] · [[Over-the-Air Updates]]
- [[Concepts MOC]]
