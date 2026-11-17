---
tags: [concept]
---
# Validation

Validation checks that input is acceptable before acting on it: required fields, types, ranges, formats, and business rules. Do it at the edge (UI for fast feedback, server for authority) and fail early with clear messages. Client validation is for usability; server validation is for safety, because a client can be bypassed.

Schemas (zod on the frontend, Bean Validation on the backend) keep rules declarative. Rules that need the database ("is this account locked?") cannot be annotations and live in services.

## In this codebase
Validation happens in layers, cheap and static first, stateful last.
- **Bean Validation on DTOs.** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/dto/tgmdpweb/projectAcceptance/PoCategoryUpdateRequestDto.java:14-22` uses `@NotBlank(message = "taskId is required")` and friends; the controller adds `@Valid @RequestBody` (`PoCategoryController.java:93`). Failure raises `MethodArgumentNotValidException`, which `config/GlobalExceptionHandler.java:45` turns into a 400 with per-field `errors[]`.
- **Path/query variables.** `PoCategoryController` is `@Validated` (`:42`), so `@PathVariable @NotBlank @Size(max = 30) String contractNo` (`:82`) is checked too. That route throws `ConstraintViolationException`, a different exception with its own handler. Two annotations, two exception types.
- **Custom constraints for legacy rules.** `validation/AtLeastOneNotBlank*.java` is a class-level constraint ("at least one of these fields") used on `dto/nwisads/tstSiteMeet/SiteMeetAddRequestDto.java:12-15` with message 至少擇一輸入。, a rule the legacy form had. `validation/CalendarDate*.java` rejects `2026-02-31` in Java instead of letting DB2 `to_date` fail later (its javadoc says exactly that).
- **Service-level rules.** State-dependent checks (captcha valid and unused, account locked, OTP expired) throw typed exceptions in `service/tgmdpvendor/TgmdpvendorAuthService.java` and are translated by controller or global handlers.
- **Frontend (zod).** `tgmdp_web_frontend/src/features/mobile-device-management/key-notification-management-operation/schema/notificationManagementCreateSchema.ts` defines the form with `z.object({...})` and Chinese messages (`z.string("請選擇系統")`), and a helper that rewrites `' - < > " &` to full-width characters so they cannot form HTML/SQL fragments.
- **Legacy.** Validation was JavaScript in the JSP, with no declarative schema: `legacy/tgmdp_vendor_legacy/TGMDP_VENDOR/TGMDPVENc001/TGMDPVENc001Q1.jsp:271-283` (`chk_data`) concatenates `errMsg += "庫房為必填\n"` and alerts. Nothing stopped a direct POST from skipping it, which is why the server rules matter.

## Why it matters
Bad input that gets through becomes bad data that is expensive to fix, and in a shared legacy DB the bad data is visible to other systems.

## Check yourself
What exception does a blank `@NotBlank` request-body field raise, and what exception does a too-long `@PathVariable` raise? Why are both handled in the global advice?

## Related
- [[Defensive Programming]]
- [[Error Handling]]
- [[Serialization and DTOs]]
- [[Authorization and RBAC]]
- [[tgmdp_backend Error Handling and Validation]]
- [[Fake ESB Internals]]
- [[tgmdp_fake_esb]]
- [[Why a Fake ESB Exists]]
- [[What an ESB Is]]
- [[Vendor Portal State and Forms]]
- [[Concepts MOC]]
- [[System Overview]]
