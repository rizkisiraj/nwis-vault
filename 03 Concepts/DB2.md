---
tags: [concept]
---
# DB2

DB2 is IBM's relational database. It follows SQL but has its own dialect and strictness: schema-qualified objects, user-defined functions resolved by a function path, and strict parameter typing that raises SQLCODE errors such as -302, -420 or -440 when types or names do not match. Errors are identified by SQLCODE and SQLSTATE rather than free text. When something fails against DB2, the SQLCODE is the search key.

## In this codebase
- **The driver and dialect.** `tgmdp_backend/build.gradle:231,244` declare the IBM `com.ibm.db2:jcc` driver (two versions are listed; Gradle resolves to the higher one) and each profile file sets `spring.jpa.database-platform=org.hibernate.dialect.DB2Dialect` (e.g. `application-uat.properties:3`).
- **Pool wiring.** `config/DataSourceConfig.java:24-37`: for `local`/`test`, an explicit Hikari pool (max 10, `:72`) wrapped in `P6DataSource` so every SQL is logged; `DB_SSL_ENABLED` switches on `sslConnection` (`:69`). Other profiles rely on Spring Boot reading `spring.datasource.url=${UAT_DB_URL}`. `tgmdp_jobs` is different: two datasources, one for Quartz tables and one for app tables (`config/AppDataSourceConfig.java:29-36`, `QuartzConfig.java:27`).
- **-302: value too long for the inferred type.** DB2 types a `?` from the other side of the comparison, so binding a longer string fails. Fix: `CAST(:siteId AS VARCHAR(50))`, with the reason in a comment (`repository/tgmdpweb/common/CommonSiteRepository.java:15-21`).
- **-420: empty string into a number.** `repository/SubcontractorRepository.java:744-747`: "COM_ID and VENDOR_ID are DECIMAL columns, empty strings cause SQLCODE=-420", so blanks are converted to `null` in Java before binding. Legacy-style callers often hand over `""` for "no value"; the Java layer has to translate it.
- **-440: unqualified function.** User-defined functions resolve through the function path, not `currentSchema`, so calls are always qualified: `ACPT.ACPT_IS_CTI_SIGNER_WEB(...)` (`repository/tgmdpweb/projectAcceptance/audit/AuditCtiMstRepository.java:105`).
- **DB2-only SQL** the code relies on: `OFFSET ... FETCH NEXT` (`CSmrInterfaceLogCustomRepositoryImpl.java:96`), sequences (`TRP.TRP_PTRLLIST_SEQ`, `model/TrpPtrllistMst.java`). Legacy equivalent: `DBDAO.searchDB2` with `ROW_NUMBER()` (`legacy/tgmdp_vendor_legacy/src/com/sys/integration/DBDAO.java:264-278`); the same class also has `searchORACLE`/`searchMSSQL`, a trace of the monolith once supporting other databases.
- **Team memory.** The longer write-ups of these bugs (and `-811`, the one bad row that broke the audit list) are in [[tgmdp_backend Persistence and DB2]].

## Why it matters
Many hard bugs here are DB2 typing or naming rules, not application logic. Reproduce with bind parameters: some of these errors do not show in a plain SQL console.

## Check yourself
The same SQL works in a DB2 console but throws -302 from the app. What differs between the two, and what do you add to the query?

## Related
- [[SQL and Relational Databases]]
- [[ORM and JPA]]
- [[Database Transactions]]
- [[Configuration and Environments]]
- [[How Jobs Read and Write Data]]
- [[Legacy Data Access with DBDAO]]
- [[Legacy TGMDP Monolith]]
- [[Legacy Systems]]
- [[Concepts MOC]]
- [[System Overview]]
