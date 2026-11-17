---
tags: [pattern]
---
# Service Layer

Up: [[Design Patterns MOC]]

## The idea
Put business rules and use-case orchestration in a service class that sits between controllers and repositories. The service owns the transaction boundary and ideally knows nothing about HTTP.

```
Controller -> Service (rules, @Transactional) -> Repository -> DB
   HTTP only      business only            SQL only
```

## Why it helps
- HTTP, business rules and SQL change for different reasons, so they live apart.
- The same service can be called from a REST controller, a SOAP endpoint or a job.
- Easy to unit test with mocked repositories.

## In this codebase
**PO category, end to end** (`tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/`):
- `controller/tgmdpweb/projectAcceptance/PoCategoryController.java:67-76` calls `poCategoryService.search(...)` and nothing else.
- `service/tgmdpweb/projectAcceptance/PoCategoryServiceImpl.java:56-66` is `@Service @RequiredArgsConstructor` and receives 8 repositories. `search` (`:130-149`) builds a `Pageable`, calls the repository, maps rows with `mapToDto` (`:421`) and wraps the result.
- The transaction boundary is declared on the service: `@Transactional` above `update(...)` (`:196-198`).
- Per-record programmatic transactions: `service/acptads/AcptApiH2Service.java:126-127` creates a `TransactionTemplate` with `PROPAGATION_REQUIRES_NEW`, used at `:275`, so one bad record in a batch does not roll back the rest.

**Honest caveat:** the service is not HTTP-free. `PoCategoryServiceImpl.java:149` returns `new ApiResponse<>(HttpStatus.OK, ...)` and the controller just does `ResponseEntity.status(response.getCode())`. The status code is decided in the service. A purist would return a result and let the controller pick the status.

**Reuse across entry points:** `service/nwisads/NwisAuthorityCheckService.java` is a service used behind SOAP/REST ESB-facing endpoints, not tied to one controller.

**Legacy vs modern.** There was no service layer. `legacy/TGMDP/src/com/prj/actions/tgmdps/TGMDPs011.java:86-123` (`browse`) does form validation, `dao.open()`, building the model, paging, `dao.rollback()`/`dao.close()` in one action method. A thin "model" class (`PoModel`) held SQL but not use-case flow. Moving to services is the main structural gain of the rewrite.

## Check yourself
Why is `@Transactional` on `update` safe, while calling a `@Transactional` method from another method of the same class would not start a transaction? (Hint: Spring proxies.)

## Related
- [[MVC]] · [[Repository Pattern]] · [[DTO Pattern]] · [[Facade]] · [[SOLID]] · [[Database Transactions]] · [[tgmdp_backend Layering and Packages]] · [[WebWork Actions vs Spring Controllers]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
