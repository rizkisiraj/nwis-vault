---
tags: [pattern]
---
# Anti-Corruption Layer

Up: [[Design Patterns MOC]]

## The idea
A translation layer that keeps an external or legacy model from leaking into your domain. Foreign names, shapes and status codes are converted at the border.

```
ESB XML (header.status 10001/20004, CHECK_STATUS 1/2) -> ACL -> NwisAuthorityCheckResponseDto
```

## Why it helps
- If the external contract changes, one class changes.
- Domain code stays in your vocabulary.

## In this codebase
**Best example: NWIS authority check** (`tgmdp_backend/.../service/nwisads/NwisAuthorityCheckService.java`).
- `callEsbAuthorityCheck` (`:144`) builds the generated SOAP request: a `CommonHeader` (`setMessageFrom("MDP")`, `setMessageTo("ESB")`, fresh `TransactionId`, `:160-166`), a `NwisAccountAuthorityCheckRequestType` with the `UserInfo.hrId` taken from the app DTO's `empNo` (`:169-175`), and calls the gateway (`nwisSoapService.nwisAccountAuthorityCheck(header, body)`, `:179`).
- It then validates and translates the response: checks the echoed header (`:191-200`), maps ESB status `10001` vs other codes and `20004` to the app's `checkStatus` "1"/"2" and `loginMsg` (`:202-213`), verifies `getNwisAccountAuthorityRes()` and `getUserInfo()`/`getFunctionList()` are populated (`:215-240`), and finally fills the app's own `NwisEsbAuthorityResponseDto` / `NwisAuthorityCheckResponseDto` (`rtnStatus`, `rtnMsg`, `checkStatus`...). The caller's contract stays "HTTP 200 with `RTN_STATUS=false`" (javadoc above `:51`), which is the legacy app's expectation, not SOAP's.
- Generated `com.taiwanmobile.esb.*` types stay inside the service layer: no class under `controller/` imports `com.taiwanmobile.esb` (checked with a grep).

**Honest limits.** This ACL is a method inside a service that also orchestrates other work (OM area lookup, node lookup in `authorityCheck`, `:91-96`), not a dedicated translator class. It is also strongly tied to legacy: many magic values ("1", "2", "10001", "20004") appear inline instead of in an enum. Good refactor target.

**Legacy-facing ACL.** The `LegacyQueryParamFilter` ([[Adapter]]) does the same job for inbound requests: legacy parameter names never reach controllers.

Not seen: a formal ACL package or interface. The idea is realised by convention.

## Check yourself
Where would you change code if the ESB renamed status `20004`? Is that one place?

## Related
- [[Adapter]] · [[Gateway]] · [[DTO Pattern]] · [[Strangler Fig]] · [[tgmdp_backend ESB Integration]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
