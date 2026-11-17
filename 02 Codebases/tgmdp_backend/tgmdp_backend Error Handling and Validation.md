---
tags: [codebase, tgmdp-backend]
---
# tgmdp_backend Error Handling and Validation

Up: [[tgmdp_backend]] · [[System Overview]]

An API's error behaviour is part of its contract. Clients branch on it, and users read it. This backend serves four kinds of clients with different expectations, so it has **four** error conventions. Learning to tell them apart is the main lesson of this note.

## The success envelope: `ApiResponse<T>`

`dto/ApiResponse.java`

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiResponse<T> {
    private int code;            // mirrors the HTTP status
    private boolean success;     // status.is2xxSuccessful()
    private String message;
    private String timestamp;
    private T data;
    private Object error = null;
    private Object meta;         // optional sidecar, e.g. a fallback record next to a page
    ...
    public static ApiResponse<Object> errorMessage(HttpStatus status, Object errorData) { ... }
}
```

`config/ResponseHelper.java` provides static factories: `success(data)`, `success(list, page)`, which builds `ApiPageResponseDto` with a **1-based** `page`, plus `error(...)` and `validationError(...)`. One predictable shape lets the frontends write a single axios response interceptor. See [[Serialization and DTOs]].

## Convention 1: throw, and let `GlobalExceptionHandler` translate

`config/GlobalExceptionHandler.java` is a `@RestControllerAdvice` with about 25 `@ExceptionHandler` methods. Code deep in a service throws a typed exception:

```java
throw new ResourceNotFoundException("Contract not found", "PO_NOT_FOUND");
```

and the advice turns it into HTTP:

| Exception | Status | Line |
|---|---|---|
| `MethodArgumentNotValidException`, `BindException` | 400 + per-field `errors[]` | `:45-91` |
| `ConstraintViolationException` | 400 | `:93-119` |
| `HttpMessageNotReadableException` (bad JSON) | 400 | `:121-138` |
| `ResourceNotFoundException` | 404 | `:140-174` |
| `UnauthorizedException` | 401 | `:176-199` |
| `AuthorizationDeniedException` (from `@PreAuthorize`) | 403 | `:259` |
| `ForbiddenException` | 403 | `:279` |
| `BadRequestException`, `ResourceConflictException`, `UnprocessableEntityException` | 400 / 409 / 422 | `:470`, `:508`, `:621` |
| `Exception` (catch-all) | 500 | `:363-391` |

**Why centralise?** Controllers stay free of `try/catch`, and every module returns the same error shape. It is the [[Interceptor and Filter Chain]] idea applied to exceptions: the cross-cutting concern lives in one place.

The catch-all deserves a close look:

```java
// GlobalExceptionHandler.java:363-391
@ExceptionHandler(Exception.class)
public ResponseEntity<Map<String, Object>> handleInternalServerError(Exception ex, HttpServletRequest request) {
    log.error("Unhandled exception", ex);
    fieldError.put("message", isOta(request) ? OtaErrorMessages.INTERNAL_SERVER_ERROR : ex.getMessage());
    boolean showStackTrace = activeProfile.equals("local") || activeProfile.equals("test");
    if (showStackTrace) { ... fieldError.put("trace", sw.toString()); }
```

- The stack trace is returned only in `local`/`test`. That is [[Environment Separation]] in practice: helpful to developers, invisible to attackers.
- But `ex.getMessage()` **is** returned in every environment. For a DB error that message can include SQL text and table names. That is an information leak. Compare the H2 inbound service, which deliberately returns a fixed message (`AcptApiH2Service.java:159-163`).

### Two exception hierarchies

Services moved into `sharedModules` throw `com.taiwanmobile.sharedmodules.exception.ResourceNotFoundException`, a **different class** with the same simple name. Without extra handlers (`:201-257`) they would fall through to the catch-all as 500s. The comment there tells the story. Watch for this whenever you share code through a library: exception types are part of the API.

### Local handlers win over global ones

`TgmdpvendorAuthController` declares its own `@ExceptionHandler`s (`:101-112`, `:155-166`, `:348-364` ...). Spring checks the controller's own handlers first. They map vendor-auth exceptions to precise statuses: invalid captcha → 400, user not found → 404, locked/deleted → 403, no mobile / not enrolled in M+ → 422, wrong OTP → 401 **with `remainingAttempts` in `data`**. Every exception class carries a machine-readable `code` (`exception/tgmdpvendor/TgmdpvendorAuthException.java`) next to the human message. The frontend switches on the code, not on Chinese text.

## Convention 2: the service returns an error envelope

`service/tgmdpweb/projectAcceptance/PoCategoryServiceImpl.java` catches everything and returns `new ApiResponse<>(HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERROR", null)`. The controller passes the code through with `ResponseEntity.status(response.getCode())`. The result reaches the client correctly, but:
- the service now knows about HTTP status codes (layering leak),
- inside `@Transactional` this can hide a rollback-only state (see [[tgmdp_backend Persistence and DB2]]),
- the root cause appears only in the log, never in a typed exception a caller could handle.

## Convention 3: legacy app contract, errors inside HTTP 200

NWIS/AMS app endpoints return `RTN_STATUS=false` + `RTN_MSG` with **HTTP 200** (`NwisAuthorityCheckService.java:47-49`). The legacy mobile app was written that way, and the rebuilt micro-apps keep the contract.

There is a cross-repo reason to prefer this for app endpoints. The team's notes record that when the backend returns a **non-2xx** status, the RN shell bridge discards the response body, so the server's error text never reaches the NWIS2 UI. A 200 with an error flag survives the bridge; a 500 with a nice message does not. Error-handling design has to account for every hop between server and user.

## Convention 4: ESB envelope

Inbound ESB endpoints reply with the ESB's own codes (`10001` success, `20002` incomplete data, `20003` exception), always HTTP 200 (`AcptApiH2Service.java:76-82`). The ESB, not a person, reads them.

> **Takeaway.** "Use proper HTTP status codes" is a good default, not a law. The client's parser defines the contract. Here, the web console gets REST-style codes; the mobile apps and the ESB get legacy envelopes. What matters is **consistency per client**. Notice that `GlobalExceptionHandler` itself is not fully consistent: it builds ad-hoc `Map`s (some with `errors`, some with `error`, and `UnprocessableEntity` without `code`/`message`) rather than `ApiResponse`.

## Validation

### Bean Validation on DTOs
```java
// dto/tgmdpweb/projectAcceptance/PoCategoryUpdateRequestDto.java
@NotBlank(message = "taskId is required")    private String taskId;
@NotBlank(message = "projectType is required") private String projectType;
```
`@Valid @RequestBody` in the controller triggers it before the method runs. A failure becomes `MethodArgumentNotValidException`, then a 400 with:

```json
{ "success": false, "code": 400, "message": "invalid parameter",
  "errors": [ { "field": "projectType", "errorCode": "NotBlank", "message": "projectType is required" } ] }
```

### Validation on path variables and query params
`PoCategoryController` is annotated `@Validated` (`:42`). That turns on **method-level** validation, so `@PathVariable @NotBlank @Size(max = 30) String contractNo` is checked too. Failures raise `ConstraintViolationException` (`GlobalExceptionHandler.java:93`). The `@Size(max = 30)` is more than tidiness: it rejects input longer than the DB column before DB2 can throw `-302`.

### Custom constraints
`validation/AtLeastOneNotBlank.java` + `AtLeastOneNotBlankValidator.java` is a **class-level** constraint ("at least one of these fields must be filled"). It reproduces a legacy form rule (至少擇一輸入) that field-level annotations can't express. Used on `dto/nwisads/tstSiteMeet/SiteMeetAddRequestDto.java`. `CalendarDate` / `CalendarDateValidator` check real calendar dates.

### Validation inside services
Many rules need the database ("is the captcha still valid?", "is this account locked?"). Those live in the service and throw typed exceptions (`TgmdpvendorAuthService.sendOtp`, `:264-341`). Layered validation, from cheap and static to expensive and stateful, is the normal shape. See [[Validation]].

## Try it yourself

1. Send `POST /api/v1/tgmdpweb/project-acceptance/po-category/X` with body `{}` (as a user with the right authorities). Predict the JSON before you run it.
2. Find an `@ExceptionHandler` in a controller other than `TgmdpvendorAuthController` and decide whether it should move to the global advice.
3. Write down which of the four conventions each of these uses: `/api/v1/tgmdpvendor/auth/otp/verify`, `/api/v1/nwis/authorityCheck`, `/ACPT_SERVICE/api/WH/API_H2`, `/api/v1/tgmdpweb/project-acceptance/po-category`.

## Questions to check understanding

- Why might returning `ex.getMessage()` in production be a security problem?
- What's the difference between `MethodArgumentNotValidException` and `ConstraintViolationException`, and which annotation enables each?
- If a mobile client discards non-2xx bodies, how should the server report a business error?

## Related

- [[tgmdp_backend]] · [[System Overview]] · [[tgmdp_backend Request Lifecycle]] · [[tgmdp_backend Persistence and DB2]] · [[tgmdp_backend Logging and Configuration]]
- [[Error Handling]] · [[Validation]] · [[HTTP and REST]] · [[Serialization and DTOs]]
- [[Interceptor and Filter Chain]] · [[DTO Pattern]] · [[Defensive Programming]] · [[Environment Separation]] · [[API Versioning]]
