---
tags: [legacy]
---

# DBDAO vs Spring Data Repositories

> Up: [[Legacy to Modern MOC]]

How the same data gets read and written, before and after the rebuild.

## Before: DBDAO

Legacy has no ORM. Every query is a hand-written SQL string in a Model class, run through one home-made wrapper, `DBDAO`. Rows come back as `ArrayList<Map<String,String>>`, with NULL turned into `""` and timestamps formatted as `yyyy/MM/dd HH:mm:ss`. The Action opens the connection, commits and closes it by hand. See [[Legacy Data Access with DBDAO]].

## After: repositories

The backend mixes several styles. Simple CRUD uses Spring Data JPA derived methods; fixed legacy SQL uses `@Query(nativeQuery = true)`; dynamic search SQL uses `NamedParameterJdbcTemplate`. See [[tgmdp_backend Persistence and DB2]].

## Side by side

| Aspect | Legacy DBDAO | Modern backend |
|---|---|---|
| SQL | Hand-built strings in `com.prj.models.*` | Native `@Query`, `NamedParameterJdbcTemplate`, a few derived methods |
| Parameters | `?` plus a param list kept in step by hand | Bound, named parameters |
| Rows | Untyped string maps | Entities or `Map` rows |
| Transactions | `open`/`commit`/`rollback`/`close` in each Action | Spring `@Transactional` ([[Database Transactions]]) |
| Paging | `PageBreak`, 1-based | `Pageable`, 0-based |

## What stayed the same

Search screens are still SQL-shaped, so the rebuild still writes much SQL by hand, much like `DeviceModel`. See [[WebWork Actions vs Spring Controllers]]. DB functions and views are reused, and DB2 quirks (`-440`, `-302`, `-420`) came with them. See [[Legacy TGMDP Monolith]] and [[DB2]].

## Related

- [[Repository Pattern]] · [[ORM and JPA]] · [[SQL and Relational Databases]]
- [[Legacy to Modern MOC]] · [[Session ButtonBean vs JWT Authorities]]
