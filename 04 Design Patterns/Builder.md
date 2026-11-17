---
tags: [pattern]
---
# Builder

Up: [[Design Patterns MOC]]

## The idea
Construct a complex object step by step through a fluent API, then call `build()`. Good when there are many optional fields or when the parts need names.

```
PoCategorySearchResponseDto.builder()
   .pagination(...).data(items).build()
```

## Why it helps
- More readable than a 20-argument constructor; each value is labelled.
- Result can be immutable.
- Nested builders mirror nested JSON.

## In this codebase
Mostly **Lombok `@Builder`** (generated, not hand-written) plus library builders.

1. **Response DTOs.** `tgmdp_backend/.../service/tgmdpweb/projectAcceptance/PoCategoryServiceImpl.java:139-147` builds a nested response:
```java
PoCategorySearchResponseDto responseDto = PoCategorySearchResponseDto.builder()
        .pagination(PoCategorySearchResponseDto.PaginationDto.builder()
                .page(pageable.getPageNumber() + 1)
                .size(pageable.getPageSize())
                .total(resultPage.getTotalElements())
                .totalPages(resultPage.getTotalPages())
                .build())
        .data(items)
        .build();
```
`PoCategorySearchResponseDto` is `@Data @Builder @NoArgsConstructor @AllArgsConstructor`, with the nested `PaginationDto` annotated the same way. The same service maps each DB row with `PoCategoryItemDto.builder().contractNo(...)...build()` (`:421-431`). Also `ApiPageResponseDto.<T>builder()...build()` in `config/ResponseHelper.java:21`, and `OtpVerifyResponseDto.builder()` in `service/tgmdpvendor/TgmdpvendorAuthService.java:596`.
2. **Cookie builder (library).** `controller/tgmdpvendor/TgmdpvendorAuthController.java:318-325` uses Spring's `ResponseCookie.from("access_token", ...).httpOnly(true).secure(cookieSecure).path("/").maxAge(Duration.ofHours(8)).sameSite(...)`, then conditionally `.domain(...)`, then `.build()` (`:327`). A builder suits this because the domain is optional: the code keeps the builder in a variable and adds it only if configured.
3. **HTTP client.** `config/HttpClientConfig.java` uses `HttpClients.custom().setConnectionManager(...).build()` and Spring's `RestClient.Builder` (`.requestFactory(...)`, `.defaultHeader(...)`, `.requestInterceptor(...)`, around `:60-71`).

**Legacy vs modern.** Legacy code had no builder: objects were filled by setters (`pb.setDao(dao); pb.setRequest(request); pb.setInit_button("browse"); pb.setModel(pomodel, "browse")`, `TGMDPs011.java:106-110`) or by `LinkedHashMap.put` (`:100-102`). That allows half-initialised objects, which `@Builder` + `build()` narrows.

## Check yourself
`@Builder` plus `@NoArgsConstructor` plus `@Data` gives mutability back. Why did the team need `@NoArgsConstructor` on a response DTO?

## Related
- [[Factory]] · [[DTO Pattern]] · [[Clean Code Naming]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
