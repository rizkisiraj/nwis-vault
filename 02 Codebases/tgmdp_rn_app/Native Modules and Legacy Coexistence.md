---
tags: [codebase, tgmdp-rn-app, legacy]
---
# Native Modules and Legacy Coexistence

> The shell is only useful if it can do things a web page cannot (scan a barcode, dial a phone, read a root-detection flag) and if it can live alongside the old Cordova apps during a migration that will take years. This note covers the custom native modules and the three coexistence mechanisms: a shared SSO session, launching legacy apps, and a Cordova compatibility shim.

Up: [[tgmdp_rn_app]] · [[System Overview]] · Legacy context: [[Strangler Fig]]

## Custom native modules

Each folder under `modules/` with an `android/` and/or `ios/` subfolder is an **Expo module**: a small Kotlin/Swift class plus a TypeScript wrapper. They are linked as local npm packages (`"data-service": "file:./modules/data-service"`, `package.json:65-66`) or autolinked.

| Module | Native capability | Reached from the web via |
|---|---|---|
| `tgmdp-camera` | In-app camera, photo with watermark/GPS | `CAPTURE_REQUEST`, `MULTI_CAPTURE_REQUEST` |
| `tgmdp-scanner` | Barcode / QR scanning | `SCAN_REQUEST`, `MULTI_SCAN_REQUEST`, `MRM_SERIAL_SCAN_REQUEST` |
| `tgmdp-call-phone` | Place a phone call | `tel:` links intercepted by the WebView (`useWebViewScreen.security.ts`, `launchPhoneCall`) |
| `tgmdp-reminder` | Local alarm/reminder | `REMINDER_REQUEST` (ports legacy AMS2 `doClock`, cited in `services/bridge/index.ts:61-62`) |
| `tgmdp-biometric` | Fingerprint / face login | shell only, through `services/platform/biometric/index.ts` (login and biometric binding) |
| `tgmdp-exit-app` | Close the app | shell only, through `utils/exit-app.ts` |
| `data-service` | **Android bound service** that shares SSO data with other apps (AIDL) | legacy apps, not guests |
| `do-legacy-things` | Token, intent and legacy helper utilities | shell |
| `device-integrity` | Plain Android Gradle library: root detection, blocks rooted devices (used by `MainActivity`) | shell |
| `okhttp-brotli-twm` | Brotli support for the OkHttp stack | shell |

Each module has its own unit tests (`android/src/test/...`). The JS side has a `.web.ts` stub where needed, so the module can be imported in Jest without native code.

Why native modules instead of a plugin from npm? Several modules port the exact behaviour of a legacy Cordova plugin (wording of a permission prompt, scan modes, reminder semantics) so that field staff see no change. Parity with legacy is a hard requirement throughout this workspace. See [[Adapter]]: each module adapts a platform API to the bridge's message contract.

## Coexistence 1: one session shared with legacy apps

Legacy apps (the Cordova AMS2, MRM, BTS...) read the logged-in user from a shared **USERFILE**. The shell writes the same file so that a user who signs in once can open a legacy app without signing in again.

- **Android**: `modules/data-service` exposes an AIDL interface that other apps bind to:

```java
// tgmdp_rn_app/modules/data-service/android/src/main/aidl/com/TGMDP2/Service/IDataService.aidl
interface IDataService {
    boolean putData(String key, String value);
    String getData(String key);
    void deleteData(String key);
}
```

- **iOS**: apps share a container through an **App Group** (`group.tgmdp.sso`, `app.config.js:69,106`) and talk through the `TGMDPSSO` URL scheme.

`AbsoluteSessionManager` treats secure storage and USERFILE as one logical write (`services/auth/session/AbsoluteSessionManager.ts:1-11`), and it can even *extend* the shell session when a legacy app has refreshed USERFILE in the meantime. Two processes that both write the session is a distributed-state problem in miniature; the "session generation" counter and the reconcile step exist to keep them consistent. See [[Authentication and Sessions]] and [[Concurrency and Threads]].

## Coexistence 2: launching a legacy app

When a menu module is not migrated (no URL configured), or the user picks a `_LEGACY` variant, the menu calls `launchLegacyModule` (`features/menu/hooks/actions/menu-screen.launch-actions.ts:152-174`). It goes through one gate:

```ts
// tgmdp_rn_app/services/auth/session/LegacyLaunchCoordinator.ts:1-7 (header)
/**
 * Single pre-launch gate for every installed legacy application.
 *
 * Native launch is attempted only after pending cleanup is resolved, React
 * Native is foreground-active, and canonical session/USERFILE reconciliation
 * succeeds. The coordinator returns typed outcomes and never displays UI.
 */
```

The coordinator returns `FAILED` (app not installed) or `BLOCKED` (with a reason) and lets the menu decide what to show. A service that returns typed outcomes and never shows UI is easy to test and reuse: [[Separation of Concerns]].

On iOS, checking whether a legacy app is installed needs its URL scheme listed in `LSApplicationQueriesSchemes`; the `withIOSQuerySchemes` plugin adds `TGMDPAMS2`, `TGMDPMRM` and the others (`app.config.js:185-211`). Its comment explains a nasty failure mode: forget the `.debug` suffix and every legacy tile silently behaves as "not installed".

## Coexistence 3: a Cordova shim for unported pages

Some web content still calls `cordova.exec(success, fail, service, action, args)`, the Cordova plugin API. Instead of rewriting it first, the shell injects a fake `window.cordova` that turns each call into a bridge message:

```js
// tgmdp_rn_app/core/communications/cordovaShim.template.ts:20-30 (inside the injected string)
window.cordova = {
    platformId: 'android',
    version: '10.0.0',
    exec: function(success, fail, service, action, args) {
        var callbackId = 'cb_' + Date.now() + '_' + Math.random().toString(36).slice(2, 11);
        callbacks[callbackId] = { success: success, fail: fail };
        window.ReactNativeWebView.postMessage(JSON.stringify({ type: 'CORDOVA_EXEC', payload: { /* ... */ } }));
    }
};
```

`features/webview/cordovaExec.handler.ts` answers those calls. One of them, `doCheckUserFile`, returns the shared USERFILE including the auth token, so `useWebViewScreen.security.ts` first verifies that the message came from the module's own origin. This is an [[Anti-Corruption Layer]] in the most literal sense: old code keeps its old API, and a translation layer maps it onto the new runtime.

## Cross-module navigation

Legacy apps opened one another (AMS opening MRM's material request page). The shell keeps that ability with `OPEN_MRM`, `OPEN_BTS`, `OPEN_EXTERNAL_APP` and `OPEN_EXTERNAL_URL` messages (`types/bridge/messageType.ts:135-142`, `features/webview/openMrmModule.ts`, `openExternalApp.ts`). `OPEN_EXTERNAL_URL` exists because WKWebView blocks `window.open`, so the guest asks native to open Maps or a portal.

## Questions to check understanding

1. Why does the shell bother to write USERFILE at all if every module will eventually be a WebView?
2. What could a malicious page do if `doCheckUserFile` did not check the message origin?
3. Which of the three coexistence mechanisms could be deleted first once all modules are migrated, and which last?

## Related

- [[tgmdp_rn_app]] · [[System Overview]] · [[Mobile Architecture Overview]]
- [[Bridge Engine and Message Protocol]] · [[Bridge API Request Flow and Auth]] · [[Hybrid Shell Architecture Trade-offs]] · [[RN Shell Build and Environments]]
- Concepts: [[Hybrid Mobile Apps]] · [[WebView and Native Bridge]] · [[Authentication and Sessions]] · [[Concurrency and Threads]]
- Patterns: [[Strangler Fig]] · [[Anti-Corruption Layer]] · [[Adapter]] · [[Facade]]
- Practices: [[Separation of Concerns]] · [[Defensive Programming]] · [[Unit Testing]]
- New: Single Sign-On · Inter-Process Communication
