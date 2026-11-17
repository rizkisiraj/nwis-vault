---
tags: [concept]
---
# Database Transactions

A transaction groups several database operations into one all-or-nothing unit. ACID: Atomic (all or none), Consistent (rules hold), Isolated (concurrent work does not interfere), Durable (committed data survives a crash). If anything fails you roll back and the database looks as if nothing happened.

In Spring, `@Transactional` wraps a method in a transaction; an unhandled `RuntimeException` triggers rollback. Swallowing an exception inside can still cause a rollback-only failure at the end (`UnexpectedRollbackException`).

## In this codebase
- **Default: one service call = one transaction.** `PoCategoryServiceImpl.update` (`.../service/tgmdpweb/projectAcceptance/PoCategoryServiceImpl.java:197-198`) is `@Transactional` and changes `ACPT_PO_MST` and `ACPT_CTI_MST` together. `@Transactional` appears on service methods, not controllers or repositories.
- **A failure that must still be saved.** `service/tgmdpvendor/TgmdpvendorAuthService.java:445-452`:
```java
@Transactional(noRollbackFor = {
        TgmdpvendorAuthInvalidOtpException.class,
        TgmdpvendorAuthOtpExpiredException.class,
        TgmdpvendorAuthAccountLockedException.class, ... })
public OtpVerifyResponseDto verifyOtp(...)
```
A wrong OTP throws (so the caller gets 401), but the incremented failure counter must commit, otherwise brute force would never lock the account. A business rule shaping transaction semantics. Reads use `@Transactional(readOnly = true)` (`:188`).
- **One transaction per record in a batch.** `service/acptads/AcptApiH2Service.java:102-127` builds a `TransactionTemplate` with `PROPAGATION_REQUIRES_NEW` and runs each ESB record inside it (`:275`), so one bad record cannot roll back the good ones already committed. The template (programmatic) is used instead of the annotation because the boundary sits inside a loop.
- **The swallowed-exception trap, recorded in a comment.** `service/acptads/CtitempproAcptService.java:883-885` explains that an empty `CTIM_ID` once produced `ctim_id='null'`, DB2 `-420`, and "tx marked rollback-only -> 500". Many services (e.g. `PoCategoryServiceImpl.update`, `:256-260`) `catch (Exception e)` and return an `INTERNAL_ERROR` envelope instead of rethrowing; inside `@Transactional` that is only safe if nothing already marked the transaction rollback-only.
- **Legacy contrast.** There is no annotation. `legacy/tgmdp_vendor_legacy/src/com/sys/integration/DBDAOCon.java:92` opens the connection with `con.setAutoCommit(false)`, and `commit()`/`rollback()` (`:153`, `:169`) are called by hand. Each action wraps work in try/catch/finally: `catch` -> `dao.rollback()`, `finally` -> `dao.close()` (`TGMDPVENc002.java:64-84`). Forget one branch and you leak a connection or a half-written change.

## Why it matters
Without correct boundaries you get half-saved records that are painful to repair, and in a system where other legacy apps share the DB, repair is manual.

## Check yourself
Why does `verifyOtp` need `noRollbackFor`, and what would an attacker gain if it were missing?

## Related
- [[ORM and JPA]]
- [[SQL and Relational Databases]]
- [[Concurrency and Threads]]
- [[Error Handling]]
- [[tgmdp_backend Persistence and DB2]]
- [[How Jobs Read and Write Data]]
- [[Job Reliability - Idempotency, Retries, Locking and Failure]]
- [[Concepts MOC]]
- [[System Overview]]
