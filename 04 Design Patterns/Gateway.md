---
tags: [pattern]
---
# Gateway

Up: [[Design Patterns MOC]]

## The idea
One class represents an external system and holds its endpoint, credentials, timeouts and protocol. The rest of the app calls a plain method.

```
nwisSoapService.nwisAccountAuthorityCheck(header, body)   // hides SOAP, WSS, TLS
```

## Why it helps
- Swap or mock the external system in one place.
- Protocol details don't leak into business code.

## In this codebase
1. **SOAP gateway.** `tgmdp_backend/.../service/nwisads/NwisSoapService.java:46` is the only class that talks to the NWIS ESB endpoint. Its constructor takes `SoapClientUtility` (`:63`). `getPort()` (`:67-80`) builds a CXF client for the generated `ESBPortType` (forcing SOAP 1.2), attaches a WS-Security handler holding the username/password (`:86-88`) and applies timeouts/TLS through `soap.configurePort(...)` (`:77`). The public methods are one-liners: `nwisAccountAuthorityCheck(...)` (`:91`) returns `getPort().nwisAccountAuthorityCheck(header, body)`. Callers such as `NwisAuthorityCheckService` never see a conduit or a password. Sibling gateways: `service/amsads/AmsSoapService.java`, `AmsPatrolSoapService`, `service/ftsads/FtsSoapService.java`, and `client/MetadataSoapClient.java` (endpoint and credentials from `@Value` properties, `:25-28`).
2. **REST gateway.** `service/tgmdpweb/projectAcceptance/AcptExternalCallbackServiceImpl.java:16-44` wraps two external ACPT APIs behind `notifyReject(...)`. Endpoints come from `@Value("${acpt.api.d2.endpoint:}")` / `c1`; the method skips quietly if config is blank, fetches a token from D2 (`restClient.post().uri(d2Url)...body(Map.class)`, `:39-44`), then posts to C1 (`:60`). The business code just calls `notifyReject`.
3. **Faking the other side.** Because each external system sits behind one class, a stand-in ESB (`tgmdp_fake_esb`) can replace it by changing one base URL (see [[Why a Fake ESB Exists]]).

**Gateway vs Facade vs ACL.** A gateway owns *where and how to reach* a system. A [[Facade]] simplifies a library. An [[Anti-Corruption Layer]] translates *models*. In this repo the gateway returns the generated SOAP type unchanged; translation happens one layer up.

**Legacy.** I did not trace how the legacy monolith called external systems, so I make no claim about a legacy gateway layer. Legacy DB access had its own wrapper (`DBDAO`, see [[Legacy Data Access with DBDAO]]), which is a gateway to the database rather than to an ESB.

## Check yourself
If the ESB host changes, how many source files in `tgmdp_backend` should you have to edit for NWIS calls? Why?

## Related
- [[Facade]] · [[Adapter]] · [[Anti-Corruption Layer]] · [[Feature Flags and Mock Toggles]] · [[tgmdp_backend ESB Integration]] · [[What an ESB Is]] · [[Vendor Portal API Layer]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
