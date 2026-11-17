---
tags: [concept]
---
# WebView and Native Bridge

Up: [[Concepts MOC]]

## The idea

A WebView is a browser engine embedded in a native app. It runs web pages but is sandboxed, so it cannot reach the camera, files or secure storage on its own. A native bridge is the channel that lets the page ask native code to do those things.

Bridges are usually message passing: JavaScript posts a JSON message, native code handles it and replies. Because the page is code with real power behind it, the protocol needs validation and versioning.

Key points:

- **Request and response**: each message carries an id so replies match calls.
- **Allow-list**: only named native methods can be called.
- **Serialization**: everything crossing the bridge becomes JSON.
- **Timeouts and errors**: native work can fail or hang, so both sides handle it.

## In this codebase

- **Web side sends** (AMS): `tgmdp-app-ams/src/utils/bridge/core.ts:80-96` posts a JSON string through `window.ReactNativeWebView.postMessage`, with a fallback to `window.webkit.messageHandlers.ReactNativeWebView` for iOS. `utils/bridge/api.ts:188` builds `type: 'API_REQUEST'`; native calls use `type: 'NATIVE_REQUEST'` (lines 265, 351). Timeouts are named constants (`api.ts:16-21`: 30 s API, 10 s info, 60 s module update). Services wrap this as `bridgeClient` (`src/services/api/bridge.ts`), which rejects if the bridge is missing. See [[AMS Bridge Client]].
- **Shell side receives** (vendor shell): `tgmdp_vendor_rn/core/bridge/application/WebViewBridgeMessageHandler.ts` parses the message, requires a valid `type`, then branches. `API_REQUEST` goes to the registered API handler; `NATIVE_MODULE_REQUEST` requires non-empty `module` and `action` and looks the module up in a registry, returning `NATIVE_MODULE_NOT_IMPLEMENTED` if unregistered. Anything else returns `UNKNOWN_MESSAGE_TYPE`. Every reply carries the original `requestId` so the page can match it. The registered set is built in `hooks/useWebViewBridge.ts:73,121-122` (an effective allow-list). See [[Bridge Engine and Message Protocol]].
- **Injection of the client**: `core/bridge/infrastructure/buildBridgeClientScript.ts` generates JS injected before the page boots; it defines `window.__nativeBridgeClient` (line 57) and the response/event callbacks (lines 101, 115), and errors if `window.ReactNativeWebView` is absent.
- **Native handlers**: nine modules (camera, scanner, storage, auth, pdf, ...) in `core/native-module-router/infrastructure/`; see [[Native Modules and Legacy Coexistence]]. The API path adds the bearer token on the native side (`core/api-proxy/infrastructure/AxiosApiClient.ts:68-71`), so the page never sees it.
- **Pooling and loading** of WebViews in the big shell: [[WebView Pool and Bundle Loading]].

## Legacy vs modern

Cordova exposed native features as plugins that the page called directly through `cordova.js` (`cordova_plugins.js` in the legacy `www`). The RN design instead uses one narrow, id-matched JSON channel, which is easier to validate and test, at the cost of writing handlers. A Cordova shim in `tgmdp_rn_app/core/communications/cordovaShim.template.ts` keeps some old calls working.

## Why it matters

Every device feature the web layer uses goes through the bridge, so its contract is the most sensitive interface in the app.

## Check yourself

What happens in the vendor shell if the page sends `{type: 'NATIVE_MODULE_REQUEST', module: 'shell', action: 'x'}`?

## Related

- [[Hybrid Mobile Apps]] · [[Serialization and DTOs]] · [[Validation]] · [[Error Handling]] · [[Adapter]] · [[Facade]]
- [[Concepts MOC]]
