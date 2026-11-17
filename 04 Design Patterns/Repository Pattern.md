---
tags: [pattern]
---
# Repository Pattern

Up: [[Design Patterns MOC]]

## The idea
A repository is a collection-like object that hides how data is stored. Callers ask for data by intent (`findByUserId`, `browse(criteria, pageable)`) and never write SQL or know whether JPQL, native SQL or JDBC runs underneath.

```
interface PoCategoryBrowseRepositoryCustom {
  Page<Map<String,Object>> browse(Criteria c, Pageable p);
}
// service depends on the interface, not on SQL
```

## Why it helps
- Services are testable by mocking one interface.
- You can change query technique (JPQL to native to JDBC) without touching business code.
- Persistence quirks (here: DB2) stay in one layer.

## In this codebase
Three styles of repository sit side by side in `tgmdp_backend`:

1. **Spring Data interface.** `repository/acptads/AcptWorkTypeMstRepository.java` is `@Repository interface ... extends JpaRepository<AcptWorktypeMst, String>` with `@Query(..., nativeQuery = true)` methods such as `getAcptProjectType()`. No implementation class; Spring generates it.
2. **Hand-written JDBC behind an interface.** `repository/tgmdpweb/projectAcceptance/PoCategoryBrowseRepositoryCustom.java:12` declares `Page<Map<String,Object>> browse(PoCategorySearchCriteria, Pageable)`. `PoCategoryBrowseRepositoryImpl.java:17-26` implements it with `NamedParameterJdbcTemplate`, builds a dynamic `WHERE` in a `StringBuilder`, and runs a count query (`:63`). The service only calls `poCategoryBrowseRepository.browse(criteria, pageable)` (`PoCategoryServiceImpl.java:133`). This is the "intent, not SQL" idea at work; dynamic filters are awkward in JPQL, so JDBC was chosen and nobody upstream noticed.
3. **Leak to watch.** `PoCategoryServiceImpl.java:66` injects `PoCategoryBulkUpdateRepositoryImpl` (the concrete class), not its `...Custom` interface, which weakens mocking and the abstraction.

Tests rely on the interface: `PoCategoryServiceTest.java:47` mocks `PoCategoryBrowseRepositoryCustom`.

**Legacy vs modern.** Legacy had no repositories. `legacy/TGMDP/src/com/sys/integration/DBDAO.java` is a generic SQL runner: callers do `dao.setCommandSQL(sql)`, `dao.setParam(...)`, `dao.search()` and get an `ArrayList` of `LinkedHashMap` rows (`makeTO`, `DBDAO.java:612-622`). SQL text lives in model classes like `PoModel.java:28-96`, so every caller knows SQL. See [[Legacy Data Access with DBDAO]] and [[DBDAO vs Spring Data Repositories]].

## Check yourself
`browse` returns `Map<String,Object>` rows, not entities. What does that tell you about why this query is not a Spring Data method?

## Related
- [[Service Layer]] · [[DTO Pattern]] · [[MVC]] · [[Factory]] · [[ORM and JPA]] · [[tgmdp_backend Persistence and DB2]] · [[tgmdp_backend Design Patterns Spotted]] · [[Separation of Concerns]] · [[Design Patterns MOC]]
