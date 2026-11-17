---
tags: [concept]
---
# ORM and JPA

An ORM (object-relational mapper) maps database rows to objects so you work with entities rather than hand-written result sets. JPA is the Java standard for it; Hibernate is the usual engine and Spring Data JPA adds repositories whose methods become queries. Entities carry annotations for table and column mapping, and the framework produces the SQL.

Pitfalls: the N+1 query problem, lazy loading outside a transaction, and surprising generated SQL. Native or hand-written queries are still used for complex or database-specific work.

## In this codebase
The rebuilt backend uses JPA, but as a **mapping and CRUD tool over a database it did not design**, not as a way to avoid SQL.
- **Entities mirror legacy tables, schema included.** `sharedModules/src/main/java/com/taiwanmobile/sharedmodules/model/acptads/AcptAccountMst.java` is `@Entity @Table(name = "ACPT_ACCOUNT_MST", schema = "ACPT")` with plain `@Column` fields. Audit columns (`create_date`, `update_id` ...) live in a `@MappedSuperclass`, `BaseEntityAcpt.java`; its comment records a real bug: without `@MappedSuperclass`, JPQL on `c.updateDate` fails at startup with `UnknownPathException`. About 430 files carry `@Entity`.
- **Relationships are almost absent.** `@OneToMany`/`@ManyToOne` appear in only 3 files (e.g. `CSetMst.java:52`, `CSetDtl.java:36`, both `LAZY`). Legacy tables mostly have composite or no surrogate keys, so ~100 files use `@IdClass`/`@EmbeddedId` (`model/BTemplatesDtl.java:13`, `model/TrpSiteFloorDtl.java:19`). You will rarely meet the N+1 problem here; you meet "which table does this join?" instead. Sequences are mapped explicitly where the DB uses them (`model/TrpPtrllistMst.java:19-28`, `TRP.TRP_PTRLLIST_SEQ`).
- **Repositories mix generated and hand-written SQL.** `repository/acptads/AcptWorkTypeMstRepository.java` extends `JpaRepository<AcptWorktypeMst, String>` but its methods are `@Query(value = "... FROM ACPT.ACPT_WORKTYPE_MST WHERE STATUS = '1'", nativeQuery = true)`. Why native: the team ports legacy SQL verbatim to get identical results, and queries use DB2 features JPQL cannot express (see [[DB2]]). Dynamic filters use `NamedParameterJdbcTemplate` in `*RepositoryImpl` classes (e.g. `PoCategoryBrowseRepositoryImpl.java:25-52`).
- **No schema generation.** `application.properties:18` sets `spring.jpa.hibernate.ddl-auto=none` and `:11` disables Liquibase: the DB is owned by legacy systems, so Hibernate must never alter it.
- **tgmdp_jobs** reuses the same entities/repositories from `sharedModules` (`AppDataSourceConfig.java:25-27` scans `com.taiwanmobile.sharedmodules.repository`) with its own entity manager and transaction manager. **Legacy contrast:** `legacy/.../sys/integration/DBDAO.java` is the pre-ORM way: build a SQL string, `setParam(...)`, `search()` returns an `ArrayList` of rows ([[Legacy Data Access with DBDAO]]).

## Why it matters
It decides how much SQL you write by hand and where performance surprises come from. Here the answer is "a lot of SQL, copied from legacy".

## Check yourself
`AcptWorkTypeMstRepository` extends `JpaRepository` yet uses no derived-query methods. What would be lost if the team switched those to JPQL?

## Related
- [[SQL and Relational Databases]]
- [[Database Transactions]]
- [[Repository Pattern]]
- [[DB2]]
- [[tgmdp_backend Persistence and DB2]]
- [[How Jobs Read and Write Data]]
- [[Web Console Feature Trace - APP Login Logout Record Query]]
- [[Concepts MOC]]
- [[System Overview]]
