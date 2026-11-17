---
tags: [concept]
---

# Idempotency

> Up: [[Concepts MOC]]

## The idea

An operation is idempotent if doing it twice has the same effect as doing it once. "Set status to 1" is idempotent; "add 1 to the counter" and "send an email" are not.

It matters because real systems repeat things: a job is rerun, a client retries after a timeout, a callback is delivered twice. If the operation is idempotent, repeats are harmless. If not, you need a guard such as a status flag or a unique key. Idempotency is a design property of the data and the operation, not something a framework adds for you.

## In this codebase

- **Idempotency key (best example).** `SyncTicketAllData` creates a fresh `transactionId` per run (`tgmdp_jobs/.../job/impl/mdp/SyncTicketAllData.java:62`, `insertSync(syncId, transactionId)` at line 66). When the ESB calls back, `MdpSoapEndpoint` (`tgmdp_backend/.../service/mdpads/MdpSoapEndpoint.java:68-120`) looks the id up and acts only if its status is still `"0"`:

```java
if ("0".equals(syncRecord.getStatus())) {
    mdpTicketSyncRepository.updateSyncStatus(syncRecord.getSyncId(), "1");
    ...process the file...
} else {
    ch.setStatus("F"); ch.setMessage("transactionId " + transactionId + " executed");
}
```

  A duplicate callback hits the `else` and is refused. `FtsSoapEndpoint.java` (~line 79-95) does the same for FTS. See [[Trace - SyncTicketAllData ESB Round Trip]].
- **Full replace.** `SyncOrgDataService.execute` (`tgmdp_jobs/.../service/mdp/SyncOrgDataService.java:53`, `deleteAllOrgData()` at line 121) deletes the whole org tree then re-inserts it inside one `@Transactional` method, so a second run yields the same table.
- **State flag on a row.** Outbox-like rows go `STATUS '0'` to `'1'`; see [[Outbox Pattern]]. `SendSmsCheck` also flips a device SMS flag right after sending (`SendSmsCheck.java:157-163`), but if the process dies between the send and the flag update the SMS goes out twice, so it is only *mostly* safe.
- **Not idempotent by design.** Sending M+ or mail is a side effect; the guard is the status column, not the call. At-least-once is the best this gives.
- **Open risk in the app.** `tgmdp_rn_app/services/bridge/resilience/bridge-retry-policy.ts:46-58` retries the `api` category up to 3 times on `408/429/500/502/503/504` and the file contains no HTTP-method check, so a failed `POST` may be re-sent. Whether the backend dedupes it is endpoint-specific; I found no generic idempotency-key header.
- **Quartz side.** `@DisallowConcurrentExecution` on 112 job classes stops two overlapping runs of the same job, which removes one source of duplicates but not retries across runs.

Legacy vs modern: the legacy code had no explicit keys; the `transactionId` guard is a modern addition for the ESB callbacks.

## Check yourself

Why does the callback guard check `status == "0"` and then update it to `"1"` *before* parsing the file? What breaks if the order is reversed and the parse is slow?

## Related

- [[Outbox Pattern]] · [[Database Transactions]] · [[Asynchronous Request-Callback]]
- [[Job Reliability - Idempotency, Retries, Locking and Failure]] · [[HTTP and REST]] · [[Concepts MOC]]
