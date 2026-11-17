---
tags: [concept]
---
# Error Handling

Error handling decides what happens when things go wrong: catch what you can recover from, translate low-level failures into meaningful ones, and let the rest propagate to a central handler. Return consistent status codes and messages to callers, and never swallow exceptions silently. A good error tells the user what to do and tells the developer where it failed.

## In this codebase
The backend has several error conventions because it serves clients with different parsers.
- **Throw and let one advice translate.** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/config/GlobalExceptionHandler.java:38` is a `@RestControllerAdvice` with about 25 `@ExceptionHandler` methods (validation -> 400, not found -> 404, `UnauthorizedException` -> 401, `AuthorizationDeniedException` -> 403 at `:259`, catch-all `Exception` -> 500 at `:363`). Controllers stay free of try/catch and every module returns the same JSON shape (`dto/ApiResponse.java`: `code`, `success`, `message`, `timestamp`, `data`).
- **Careful with the catch-all.** `:373-381`: the message is `ex.getMessage()` and a stack trace is added only when the profile is `local` or `test`. The message is returned in every environment, so a DB error text can reach the client; some handlers swap in a fixed message for OTA endpoints (`isOta(request)`).
- **Errors inside HTTP 200 for app clients.** App DTOs such as `dto/nwisads/NwisPatrolStartResponseDto.java:19-23` carry `@JsonProperty("RTN_STATUS") Boolean status` and `RTN_MESSAGE`. This keeps the legacy mobile app's contract, and the team's notes say the RN shell discards the body of non-2xx responses (so error text would never reach the UI).
- **ESB envelope.** `service/acptads/AcptApiH2Service.java:77-81` replies with the ESB's own codes `"10001"` success, `"20002"` incomplete data, `"20003"` exception, because a machine, not a person, reads them.
- **Service returns an error value.** `service/tgmdpweb/projectAcceptance/PoCategoryServiceImpl.java:256-260` catches `Exception`, logs, and returns `new ApiResponse<>(INTERNAL_SERVER_ERROR, "INTERNAL_ERROR", null)`. Simple for the controller, but the service now knows HTTP codes, and inside `@Transactional` it can hide a rollback-only state (see [[Database Transactions]]).
- **Frontend.** `tgmdp_vendor_frontend/src/lib/client.ts:133-139` treats 401 specially (auto logout) in a response interceptor.
- **Legacy.** Every action ends the same way: `catch (Exception e) { forward = "error"; dao.rollback(); ExceptionResolver.handle(this.getClass(), e, request, dao); } finally { dao.close(); }` (`legacy/tgmdp_vendor_legacy/src/com/prj/actions/tgmdpvenc/TGMDPVENc002.java:76-84`). `sys/presentation/ExceptionResolver.java` logs, picks an `ErrorBean` (SQL errors include the SQL text), stores it in the request, and the framework forwards to an error JSP. Error handling is repeated in each action rather than centralised.

## Why it matters
Users see error handling more than any other part of the app. Here "correct HTTP status" is not always the goal: the contract is whatever the specific client can parse.

## Check yourself
Which of these returns HTTP 200 on failure: the web console, the field-engineer app, the ESB inbound endpoint? What is each client's reason?

## Related
- [[Logging and Observability]]
- [[Defensive Programming]]
- [[Validation]]
- [[Database Transactions]]
- [[tgmdp_backend Error Handling and Validation]]
- [[Vendor Portal API Layer]]
- [[Fake ESB Internals]]
- [[What an ESB Is]]
- [[WebWork Actions vs Spring Controllers]]
- [[Concepts MOC]]
- [[System Overview]]
