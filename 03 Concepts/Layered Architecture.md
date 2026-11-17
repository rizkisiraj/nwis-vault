---
tags: [concept]
---

# Layered Architecture

Layered architecture splits code into horizontal layers, each talking only to the one below: presentation (controllers, pages), business logic (services), and data access (repositories, DAOs). A change to the database code should not force changes in the UI, and business rules can be tested without HTTP.

The danger is leaky layers, such as SQL in a page or business rules in a controller.

## In this codebase

- **Modern backend: controller, service, repository, DTO.** One feature, three files with the same stem: `controller/tgmdpweb/prerequisitionmaterials/PersonalMaterialsImportController.java` (`@RestController`, only routing, permission checks and the call `service.search(...)`), `service/tgmdpweb/prerequisitionmaterials/PersonalMaterialsImportService.java` (`@Service`, holds `ACommonOrgRepository`, `TgmdpwebAuthRepository` and `PersonalMaterialsImportRepository`), and `repository/tgmdpweb/prerequisitionmaterials/PersonalMaterialsImportRepository.java` (SQL). DTOs in `dto/` separate wire shapes from entities. Packages under `controller/`, `service/`, `repository/` repeat per caller (`tgmdpweb`, `tgmdpvendor`, `amsads`, ...). See [[tgmdp_backend Layering and Packages]].
- **Shared data layer.** Entities and common repositories live in the `sharedModules` Gradle module (`tgmdp_backend/sharedModules/...`), which `tgmdp_jobs` also includes (`settings.gradle`), so the two deployables share one data-access layer.
- **Jobs.** `tgmdp_jobs` layers the same way in miniature: job class (`job/impl/mdp/SyncOrgData.java`) just calls `syncOrgDataService.execute()`; the service (`service/mdp/SyncOrgDataService.java`) owns the transaction and repositories.
- **Legacy: JSP, action, DAO.** `legacy/TGMDP`: JSP view, WebWork action class (`com.prj.login.Login`), then `com/sys/integration/DBDAO.java` for SQL. Layers existed but leaked: `Login.java:63-64` puts `UserBean` and `ButtonBean` straight into the HTTP session, and `FrontControllerFilter` reads them back. See [[Legacy TGMDP Monolith]] and [[Legacy Data Access with DBDAO]].
- **Frontend layers.** Per feature, `api/` (HTTP), `pages/` (UI), `store/` (state), e.g. under `tgmdp_web_frontend/src/features/...`; see [[Vendor Portal Folder and Feature Structure]].
- **Mobile layers.** Native shell, bridge, then embedded web layer: [[RN Shell App Structure]].
- **A leak to know about.** The controller reads `@AuthenticationPrincipal` and passes `userId` down, and the service re-resolves scope from it; permission rules are duplicated between `@PreAuthorize` strings and service logic. Not wrong, but a place to look when authority bugs appear.

## Why it matters

Knowing the layers tells you where a given bug or change belongs before you open a file.

## Related

- [[Separation of Concerns]] · [[Service Layer]] · [[Repository Pattern]] · [[MVC]] · [[Feature-Sliced Structure]]
- [[WebWork Actions vs Spring Controllers]] · [[Concepts MOC]] · [[System Overview]]
