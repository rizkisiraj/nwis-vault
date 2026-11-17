---
tags: [codebase, tgmdp-backend]
---
# tgmdp_backend Persistence and DB2

Up: [[tgmdp_backend]] · [[System Overview]]

The backend did not design its database. It **inherited** a large IBM DB2 database that legacy systems have used for many years, and other legacy apps still read and write it today. Every persistence decision here follows from that fact:

- no schema generation (`spring.jpa.hibernate.ddl-auto=none`, `application.properties:18`),
- no migrations at runtime (`spring.liquibase.enabled=false`, `:11`),
- heavy use of hand-written SQL that copies legacy queries, so results match the old system exactly.

## One datasource, many schemas

There is **one** `DataSource`. You will not find a second datasource bean or a routing datasource. `config/DataSourceConfig.java` defines one bean per profile:

```java
// config/DataSourceConfig.java:24-37
@Bean @Primary @Profile("local")
@ConfigurationProperties(prefix = "spring.datasource.hikari")
public DataSource localDataSource() {
    String dbUrl = env.getProperty("LOCAL_DB_URL");
    ...
    return new P6DataSource(createDataSource(dbUrl, dbUsername, new String(dbPasswordChars)));
}
```

- `local` and `test` profiles: an explicit HikariCP pool (max 10, min idle 2, 30 s connect timeout), wrapped in **P6Spy** so every SQL is logged (`:54-79`). `DB_SSL_ENABLED=true` turns on DB2's `sslConnection`.
- `uat`, `staging`, `production`: no custom bean. Spring Boot auto-configures from `spring.datasource.url=${UAT_DB_URL}` and so on in `application-uat.properties`. *I could not confirm whether P6Spy SQL logging is active there; it depends on the env-provided JDBC URL.*

That single connection reaches **several DB2 schemas**. A rough count of schema-qualified references in `src/main/java`: `DB2ADMIN` ~970, `ACPT` ~600, `MRM` ~240, `NWIS` ~230, `AMS` ~60. Each schema belonged to a different legacy system. That is why native SQL nearly always writes `ACPT.ACPT_PO_MST`, not `ACPT_PO_MST`: the connection's default schema covers only one of them.

## Three ways to talk to the DB, all in use

| Style | Rough count | When this repo uses it | Example |
|---|---|---|---|
| Spring Data JPA derived methods / `save()` | many | simple CRUD on mapped entities | `loginRepository.findById(...)` in `TgmdpvendorAuthService.java:476` |
| `@Query(nativeQuery = true)` on a JPA repository | ~155 files | fixed SQL copied from legacy, returns entities or `Map` rows | `repository/acptads/AcptWorkTypeMstRepository.java` |
| `NamedParameterJdbcTemplate` / `JdbcTemplate` | ~90 files | **dynamic** SQL (optional filters, `IN` lists), paging by hand | `repository/tgmdpweb/projectAcceptance/PoCategoryBrowseRepositoryImpl.java` |
| `EntityManager.createNativeQuery` | ~70 files | one-off inserts with sequences, positional params | `repository/SubcontractorRepository.java:744-760` |

**Why so much native SQL instead of JPQL?** Three reasons, all from the legacy context:
1. The legacy WebWork/JSP apps wrote raw SQL. Porting it verbatim is the cheapest way to get *identical* results, ordering and null handling.
2. The queries lean on DB2 features JPQL can't express well: user-defined functions like `ACPT.ACPT_IS_CTI_SIGNER(...)`, `OFFSET ... FETCH NEXT`, `to_char(date, 'yyyy/mm/dd')`, sequences `ACPT.ACPT_VENINTERFACE_LOG_SEQ.NEXTVAL`.
3. Many tables have no surrogate key or relationship mapping, so `@OneToMany` style navigation is not available.

The trade-off: you lose database portability and compile-time checking of column names. The DB will never change here, so the first doesn't matter. The second is covered by tests that mock `NamedParameterJdbcTemplate`, plus runtime checks against a real DB2.

### Dynamic SQL done safely

```java
// PoCategoryBrowseRepositoryImpl.java:30-51
if (hasValue(criteria.getContractNo())) {
    where.append("AND a.CONTRACT_NO LIKE :contractNo ");
    params.addValue("contractNo", "%" + criteria.getContractNo().trim() + "%");
}
if (hasValue(criteria.getProjectType())) {
    for (int i=0; i<criteria.getProjectType().size(); i++){
        where.append(i == 0 ? " and (" : " or ");
        where.append("a.PROJECT_TYPE = :projectType_" + i + " ");
        params.addValue("projectType_" + i, criteria.getProjectType().get(i));
    }
    where.append(")");
}
```

The *structure* of the SQL is concatenated (`AND ...`, numbered placeholder names), but every *value* is a bind parameter. That is the rule that prevents SQL injection. See [[SQL and Relational Databases]] and [[Defensive Programming]].

> Contrast: `PoCategoryServiceImpl.update` (`:233-239`) builds a SQL **string for the audit log** with `replaceFirst("\\?", "'" + escapeSqlValue(...) + "'")`. That string is only stored in `SYS_LOG` to mimic the legacy audit format; it is never executed. Still, notice why it needs `escapeSqlValue`: a log viewer could later be tricked into running it.

## Transactions

`@Transactional` appears roughly 900 times, nearly always on **service** methods. A few techniques are worth studying.

### 1. Default: one service call = one transaction
`PoCategoryServiceImpl.update` (`:196-198`) updates `ACPT_PO_MST` and `ACPT_CTI_MST`. Both must change together or not at all.

### 2. `readOnly = true` for reads
`TgmdpvendorAuthService.validateCaptcha` (`:188`) is marked read-only. It is a hint to Hibernate (skip dirty checking) and to the driver.

### 3. `noRollbackFor`: when a failure must still be saved
```java
// service/tgmdpvendor/TgmdpvendorAuthService.java:445-452
@Transactional(noRollbackFor = {
        TgmdpvendorAuthInvalidOtpException.class,
        TgmdpvendorAuthOtpExpiredException.class,
        TgmdpvendorAuthAccountLockedException.class, ... })
public OtpVerifyResponseDto verifyOtp(OtpVerifyRequestDto request, String clientIp) {
```
A wrong OTP **throws** (so the controller returns 401), but the incremented fail counter and the audit row must still **commit**. Otherwise brute-forcing the OTP would never lock the account. By default Spring rolls back on any `RuntimeException`; `noRollbackFor` overrides that for these specific exceptions. This is a great example of a business rule shaping transaction semantics.

### 4. `REQUIRES_NEW` per record in a batch
```java
// service/acptads/AcptApiH2Service.java:100-127
/** Each MR_LIST record is run here in its own REQUIRES_NEW transaction so a failure/validation
 *  error on one record can never roll back records already committed earlier in the batch. */
this.requiresNewTransactionTemplate = new TransactionTemplate(transactionManager);
this.requiresNewTransactionTemplate.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
```
The ESB sends a batch of material-requisition results. One bad record must not undo the good ones. That matches the legacy behaviour, which committed per step. `TransactionTemplate` is the programmatic alternative to `@Transactional`, handy when the boundary sits *inside* a loop (`:275`).

### 5. Pitfall: catching an exception inside `@Transactional`
`PoCategoryServiceImpl.update` catches `Exception` and returns `ApiResponse(500)`. Two things can go wrong:
- If the failing call went through a transactional Spring proxy (e.g. a Spring Data repository), the transaction may already be **marked rollback-only**. When the service method returns normally, the commit fails with `UnexpectedRollbackException`. The user then sees a generic 500 from `GlobalExceptionHandler` instead of the friendly message. `CtitempproAcptService.java:884-886` records exactly this: a DB2 `-420` "marked rollback-only → 500".
- If the failing call was plain JDBC with no interceptor, nothing marks the transaction. Earlier writes in the same method **commit**, leaving partial data.

Rule of thumb: inside `@Transactional`, either let the exception propagate, or call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` before returning an error.

See [[Database Transactions]].

## DB2 specifics and pitfalls

These are real bugs that cost the team time. The comments in the code are the evidence.

### SQLCODE -302: bind value too long for an inferred type
DB2 infers a parameter marker's type from what it is compared with. Compare `?` with a `VARCHAR(10)` column, bind an 11-character string, and you get `-302` (SQLSTATE 22001). The fix is to give the marker an explicit type:

```java
// repository/tgmdpweb/common/CommonSiteRepository.java:15-21
// CAST prevents DB2 SQLCODE=-302 when parameter value length exceeds column definition.
"AND (CAST(:siteId AS VARCHAR(50)) IS NULL OR UPPER(a.SITE_ID) LIKE UPPER(CAST(:siteId AS VARCHAR(50)) || '%')) "
```

The same idea fixes the "optional filter" idiom `(:x IS NULL OR col = :x)`. Without a `CAST`, DB2 cannot type a bare `:x IS NULL` at all. The team's notes also record a variant: `? <> '2'` types the marker as `VARCHAR(1)`, so binding `'2B'` throws `-302`. The fix is to `CAST` it.

### SQLCODE -420: empty string into a number
```java
// repository/SubcontractorRepository.java:744-747
// COM_ID and VENDOR_ID are DECIMAL columns — empty strings cause SQLCODE=-420.
Long comIdVal = (comId != null && !comId.isBlank()) ? Long.parseLong(comId) : null;
```
Legacy Java often passed `""` or the string `"null"`. DB2 refuses to cast those to `DECIMAL`/`INTEGER`. Convert to `null` in Java. The team's notes add that an `IN (...)` "sentinel" value must have the same type as the column, and that this only shows up with bind parameters, not in a SQL console.

### SQLCODE -440: unqualified function name
User-defined functions are resolved through the SQL **function path**, which is not the same as `currentSchema`. Calling `ACPT_IS_CTI_SIGNER(...)` unqualified fails with `-440`. The code always schema-qualifies:
```java
// repository/tgmdpweb/projectAcceptance/audit/AuditCtiMstRepository.java:105
inner.append(" AND ACPT.ACPT_IS_CTI_SIGNER_WEB(a.CTIM_ID, b.SIGNER_LEVEL, :userId) = 1");
```

### SQLCODE -811: a scalar subquery returned more than one row
The team found that one bad data row (a CTIM_ID with duplicate signer rows) made `ACPT.ACPT_IS_CTI_SIGNER` fail with `-811`. That one row broke the whole audit list query for everyone. The lesson: functions and scalar subqueries that "can only return one row" will eventually meet data that breaks the assumption.

### SQLCODE -1585 and -204
- `repository/tgmdpweb/TgmdpwebBaseStationInspectionQueryRepositoryImpl.java:454`: an `ORDER BY` was removed because sorting very wide rows needed a temp tablespace with a bigger page size (`-1585`).
- `service/nwisads/NwisReplyPatrolService.java:318`: `-204` / `42704` (object not found) is detected and handled explicitly. A table that exists in one environment may not exist in another.

### Oracle-flavoured SQL
You will see `sysdate`, `to_char(...)` and `SEQ.NEXTVAL` (e.g. `PoAcptRepository.UPDATE_SQL`, `SubcontractorRepository.java`). These read like Oracle. DB2 accepts them (`to_char` is a synonym of `VARCHAR_FORMAT`; `sysdate` normally needs DB2's Oracle-compatibility setting). *I could not verify the database's compatibility settings from the repo.*

### Leftovers from the boilerplate
`AcptPoMst.setDate` declares `columnDefinition = "timestamptz"`, which is a PostgreSQL type, and the GitLab CI test job starts a `postgres` service. Both look like leftovers from the original template. They are harmless only because `ddl-auto=none`. Don't copy them.

### Schema truth lives in the DB, not in Liquibase
`src/main/resources/db/changelog/` exists, but Liquibase is disabled. The team's notes say the changelogs have **drifted** from the real DB2 schema. When you need the true column type, query the live catalog (`SYSCAT.COLUMNS`) instead of reading the changelog.

## Try it yourself

1. Open `CommonSiteRepository.java`. Remove one `CAST` in your head and predict the SQLCODE when `siteId` is 60 characters long.
2. In `TgmdpvendorAuthService.verifyOtp`, delete `noRollbackFor` in your head. Walk through five wrong OTPs. Does the account ever lock?
3. Find another `@Transactional` method in `service/` that catches `Exception` and returns an error object. Decide whether it can leave partial writes.

## Questions to check understanding

- Why does this repo prefer `NamedParameterJdbcTemplate` for search screens but Spring Data for single-row lookups?
- What is the difference between a transaction marked rollback-only and one that was rolled back?
- Why is `REQUIRES_NEW` dangerous if the outer transaction holds a lock the inner one needs?

## Related

- [[tgmdp_backend]] · [[System Overview]] · [[tgmdp_backend Layering and Packages]] · [[tgmdp_backend Request Lifecycle]] · [[tgmdp_backend Error Handling and Validation]]
- [[DB2]] · [[SQL and Relational Databases]] · [[ORM and JPA]] · [[Database Transactions]]
- [[Repository Pattern]] · [[Template Method]] · [[Defensive Programming]] · [[Configuration and Environments]] · [[Logging and Observability]]
