---
tags: [pattern]
---
# MVC

Up: [[Design Patterns MOC]]

## The idea
Split a UI-facing feature into three roles: the **Model** (data and business rules), the **View** (what the user sees) and the **Controller** (takes input, calls the model, picks the view). Each role changes for a different reason, so you can edit one without breaking the others.

```
request -> Controller -> Model (rules, data)
                |
                +--> View (HTML / JSON)
```

## Why it helps
- Business rules stay out of templates and out of HTTP code.
- Views can be swapped (JSP page, JSON, mobile) without touching the model.
- Controllers stay thin: parse input, delegate, return.

## In this codebase
Best seen by comparing the SAME screen (PO category, task TGMDPs011) before and after.

**Legacy (WebWork + JSP).** The route is declared in `legacy/TGMDP/src/WW_XML/TGMDPs.xml:336-338` (`browse` -> `com.prj.actions.tgmdps.TGMDPs011`, result chains to `begin`, which renders `/common/Template04.jsp?page=TGMDPs011/TGMDPs011Q1`). The action `TGMDPs011.java:86-123` is the controller. It builds a `PoModel` (model, `legacy/TGMDP/src/com/prj/models/PoModel.java:28`), runs the paged query, then hands rows to the view by request attribute: `request.setAttribute("view_data", pomodel.getResultData())`. The JSP reads it back at `TGMDPs011Q1.jsp:13` (`request.getAttribute("view_data")`). The "view" is server-rendered HTML.

**Modern (Spring).** `tgmdp_backend/.../controller/tgmdpweb/projectAcceptance/PoCategoryController.java:67-76`:
```java
@GetMapping(value = {"", "/"})
@PreAuthorize("hasAuthority('TASK_TGMDPs011:browse') and hasAuthority('SYS_工程驗收')")
public ResponseEntity<ApiResponse<PoCategorySearchResponseDto>> search(
        @ParameterObject PoCategorySearchCriteria criteria, ...) {
    ApiResponse<PoCategorySearchResponseDto> response = poCategoryService.search(criteria, currentUser);
    return ResponseEntity.status(response.getCode()).body(response);
}
```
The controller is now thin (parse, authorize, delegate). The view is JSON: a `*ResponseDto` serialized by Jackson, and the real "view" moved into the React frontends. The authority string `TASK_TGMDPs011:browse` reuses the legacy task id and button id.

**Why it changed:** the legacy action did validation, DB open/close, query building, paging and error handling itself (`form.checkForm()`, `dao.open()`, `dao.rollback()`, `dao.close()` in one method). The new split gives each job a class (see [[Service Layer]], [[Repository Pattern]]).

Not classic MVC: there is no controller-to-view mapping on the server in the modern stack. It is "controller + JSON API", with the UI in a separate SPA (see [[Server-rendered JSP vs React SPA]]).

## Check yourself
In `TGMDPs011.java`, find every responsibility in `browse()` that the Spring `search` method no longer does. Where did each one go?

## Related
- [[WebWork Actions vs Spring Controllers]] · [[Front Controller]] · [[Service Layer]] · [[Repository Pattern]] · [[DTO Pattern]] · [[Container-Presentational]] · [[Separation of Concerns]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
