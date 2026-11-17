---
tags: [pattern]
---
# DTO Pattern

Up: [[Design Patterns MOC]]

## The idea
A Data Transfer Object is a plain class that carries data across a boundary (HTTP, SOAP, a layer). It is shaped for the caller, not copied from the database entity, so internal fields never leak and the contract stays stable.

```
Entity / DB row (internal)  --map-->  ResponseDto (only what the client needs)
RequestDto (validated)      --map-->  entity / command
```

## Why it helps
- Decouples the wire format from the storage model.
- Validation annotations live on request DTOs.
- Prevents accidentally exposing fields like password hashes.

## In this codebase
All under `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/dto/` (folders per system: `acptads`, `amsads`, `tgmdpweb`, ...).

- **Request DTO with validation.** `dto/tgmdpweb/projectAcceptance/PoCategoryUpdateRequestDto.java` is `@Getter @Setter @NoArgsConstructor` with `@NotBlank(message = "projectType is required")` etc. The controller takes it as `@Valid @RequestBody` (`PoCategoryController.java:93`), so bad input is rejected before the service runs.
- **Response DTO built by hand-mapping.** The repository returns raw `Map<String,Object>` rows keyed by DB column names; `PoCategoryServiceImpl.mapToDto` (`:421-431`) converts `row.get("CONTRACT_NO")` into `PoCategoryItemDto.builder().contractNo(...)`. Column names and UPPER_CASE never reach the client. `PoCategorySearchResponseDto` (`@Data @Builder`) wraps `pagination` + `data`.
- **Envelope DTO.** `ApiResponse<T>` and `ApiPageResponseDto<T>` wrap every answer with a code, message and data (`config/ResponseHelper.java:15-44`).
- **Per-endpoint request params.** `PoCategorySearchCriteria` is a DTO bound from the query string (`@ParameterObject`, `PoCategoryController.java:71`).

**Legacy vs modern.** Legacy had two weak equivalents: `*Form` classes (`legacy/TGMDP/src/com/prj/actionforms/tgmdps/TGMDPs011Form.java`, used as WebWork `ModelDriven`, with a hand-called `form.checkForm()`), and untyped rows: `DBDAO.makeTO` (`legacy/TGMDP/src/com/sys/integration/DBDAO.java:612`) returns an `ArrayList` of `LinkedHashMap`, which the JSP reads directly. Typing and validation were by convention, not by the compiler.

**Mirror on the frontend.** The TypeScript types in the React apps copy these DTO field names; changing a DTO is therefore a cross-repo change (see [[Serialization and DTOs]]).

## Check yourself
`mapToDto` takes a `Map<String,Object>`. What bug can slip through that an entity-based mapping would catch at compile time?

## Related
- [[Anti-Corruption Layer]] · [[Builder]] · [[Service Layer]] · [[MVC]] · [[Serialization and DTOs]] · [[Validation]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
