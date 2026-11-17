---
tags: [concept]
---
# SQL and Relational Databases

A relational database stores data in tables of rows and columns, with keys linking tables. SQL is the language to query it: SELECT with WHERE, JOIN to combine tables, GROUP BY to aggregate, and INSERT, UPDATE, DELETE to change data. Indexes speed up lookups; constraints (primary key, foreign key, NOT NULL) protect integrity.

Two habits matter most: pass values as **bind parameters** (never concatenate user input into SQL, which is SQL injection), and keep column types and bound value types consistent (a classic source of runtime errors, see [[DB2]]).

## In this codebase
SQL is the main way the system reads data; the team copied most queries from the legacy apps so results match exactly.
- **Dynamic filters, safely.** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/repository/tgmdpweb/projectAcceptance/PoCategoryBrowseRepositoryImpl.java:25-52` builds the WHERE clause piece by piece (`where.append("AND a.CONTRACT_NO LIKE :contractNo ")`) but every value goes through `params.addValue(...)`. For a multi-select it generates numbered placeholders (`:projectType_0`, `:projectType_1` ...). The structure is concatenated, the values never are. That is the rule that prevents injection.
- **Several schemas, one connection.** Queries name the owner schema explicitly (`FROM ACPT.ACPT_WORKTYPE_MST` in `repository/acptads/AcptWorkTypeMstRepository.java`, `MRM.MRM_WAREHOUSE_MST` in legacy). Each schema belonged to a different legacy system.
- **Paging by hand.** Modern: `repository/CSmrInterfaceLogCustomRepositoryImpl.java:96` appends `OFFSET :offset ROWS FETCH NEXT :pageSize ROWS ONLY`. Legacy: `legacy/tgmdp_vendor_legacy/src/com/sys/integration/DBDAO.java:264-278` (`searchDB2`) wraps the query in `ROW_NUMBER() OVER (order by ...)` and filters `ROW_NUM > start and <= end`, and throws if no ORDER BY was set.
- **Legacy SQL is strings plus a parameter list.** `legacy/tgmdp_vendor_legacy/src/com/prj/models/WarehouseModel.java:22-40` concatenates a base SELECT, adds `left join ... where ... vendor_id = ?` only if a company was given, then `dao.setParam(__company)` and `dao.search()`. Values are `?`-bound; `DBDAO` separately rebuilds a literal copy of the SQL (`getRealSQL()`, used at `DBDAO.java:73`) for error logging only.
- **Cast when a type cannot be inferred.** Optional filters `(:x IS NULL OR col LIKE ...)` use `CAST(:siteId AS VARCHAR(50))` (`repository/tgmdpweb/common/CommonSiteRepository.java:15-23`).

## Why it matters
All the business data lives here, so every feature eventually becomes a query, and the legacy apps still write to the same tables.

## Check yourself
In `PoCategoryBrowseRepositoryImpl`, which parts are concatenated and which are bound? Why is that split safe?

## Related
- [[DB2]]
- [[ORM and JPA]]
- [[Database Transactions]]
- [[Repository Pattern]]
- [[tgmdp_backend Persistence and DB2]]
- [[How Jobs Read and Write Data]]
- [[Legacy Data Access with DBDAO]]
- [[ams2-app-legacy]]
- [[Web Console Feature Trace - APP Login Logout Record Query]]
- [[Concepts MOC]]
- [[System Overview]]
