---
tags: [pattern]
---
# Observer and Pub-Sub

Up: [[Design Patterns MOC]]

## The idea
Subjects notify subscribers when something happens, without knowing who they are. Observer is a direct listener; Pub-Sub adds a topic or bus in between.

```
const unsubscribe = bus.subscribe(handler)
bus.publish(event)
unsubscribe()   // on cleanup
```

## Why it helps
- Producer and consumer are decoupled.
- Always return an unsubscribe to avoid leaks.

## In this codebase
The pattern is **prominent in the frontends, thin in the backend**.

1. **AMS bridge message bus (clearest).** `tgmdp-app-ams/src/utils/bridge/core.ts:52-60`:
```ts
export function subscribeToMessages(fn: MessageSubscriber): () => void {
    subscribers.add(fn);
    return () => subscribers.delete(fn);
}
export function dispatchInbound(msg: NativeMessage): void {
    subscribers.forEach(fn => { fn(msg); });
}
```
The React Native shell posts messages into the WebView; `dispatchInbound` fans them out to every subscriber. `utils/bridge/api.ts:460-495` `initBridgeResponseSubscriber()` subscribes once (guard flag `subscribed`, `:457`), routes `API_RESPONSE` and native-op responses by `type` to `handleApiResponse` / `handleNativeResponse`, and returns a cleanup that unsubscribes. `src/App.tsx:127-128` calls it inside `useEffect` and keeps the returned cleanup. Why: the bridge is asynchronous (request sent now, answer arrives later as an unrelated message), so a listener plus a requestId match is the natural shape. See [[AMS Bridge Client]] and [[Asynchronous Request-Callback]].
2. **CXF logging listener (backend, weak Observer).** `tgmdp_backend/.../logging/EsbLogEventSender.java:10` implements CXF's `LogEventSender`; its `send(LogEvent event)` (`:20`) is called for each SOAP request/response/fault (switch on `REQ_OUT/RESP_IN/...`). It is handed to CXF's `LoggingInInterceptor`/`LoggingOutInterceptor` in `CxfConfig.addEsbLogging(...)` (called e.g. at `config/CxfConfig.java:111`). This is closer to a callback sink than a full Observer: there is one receiver, no subscribe/unsubscribe list.
3. **Not present.** I found no `ApplicationEvent`, `@EventListener` or `publishEvent` in `tgmdp_backend` or `sharedModules`. Zustand stores (e.g. `useUIStore`) also behave as observable state, but that is covered in [[Store Pattern]].

**Legacy vs modern.** The legacy server-rendered stack has no event bus; flow is request/response and `chain` results in `xwork` config. Cordova apps used DOM events, which I did not trace.

## Check yourself
In `initBridgeResponseSubscriber`, what would happen on every React re-render if it did not return a cleanup and had no `subscribed` guard?

## Related
- [[Store Pattern]] · [[Custom Hooks]] · [[Interceptor and Filter Chain]] · [[WebView and Native Bridge]] · [[Design Patterns MOC]]
