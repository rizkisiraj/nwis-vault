---
tags: [pattern]
---
# Facade

Up: [[Design Patterns MOC]]

## The idea
A Facade offers one simple interface over a complicated subsystem. Callers make one call; the facade handles the many fiddly steps behind it.

```
client.configurePort(port, url, timeouts)
   -> Facade { set endpoint; get conduit; set timeouts; install TLS; return port }
```

## Why it helps
- Hides library details such as CXF conduits and TLS settings.
- One place to fix subsystem quirks.
- Smaller surface for callers to misuse.

## In this codebase
- **SOAP client setup.** `tgmdp_backend/.../utils/SoapClientUtility.java:42` `configurePort(port, endpointUrl, connectionTimeout..., ...)` casts to `BindingProvider` and sets the endpoint (`:46-49`), gets `ClientProxy.getClient(port)` and its `HTTPConduit` (`:58-62`), applies an `HTTPClientPolicy` (timeouts, `:63`) and `TLSClientParameters` (`:73`). Variants `configurePortWithWsSecurity` (`:87`) and `configurePortWithBtsContext` (`:102`) add more. Callers just write `return soap.configurePort(port, nwisEsbEndpoint, 130000, 130000, null)` (`service/nwisads/NwisSoapService.java:77`). Eleven files reference the utility (itself included), so one fix lands everywhere. (It is also a utility: the trust-all `X509TrustManager` inside at `:33-39` is a security smell worth reading.)
- **Legacy DB facade.** `legacy/TGMDP/src/com/sys/integration/DBDAO.java` hides JDBC: `open()`, `setCommandSQL`, `setParam`, `search()`, `close()` replace `PreparedStatement`/`ResultSet` handling (`DBDAO.java:59-85`).

**Not really a Facade, despite the earlier list:** `ResponseHelper` (`config/ResponseHelper.java:15-64`) only builds `ApiResponse` objects with fixed messages. That is a [[Factory]] (static factory methods), not a simplification of a subsystem.

Spring's own `RestClient` is also a facade over HTTP clients, but the project mostly adds its own wiring (`HttpClientConfig`).

## Check yourself
What would break if `SoapClientUtility` were removed and each SOAP service configured its own conduit? List the files you would need to touch.

## Related
- [[Gateway]] · [[Adapter]] · [[Anti-Corruption Layer]] · [[Service Layer]] · [[tgmdp_backend ESB Integration]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
