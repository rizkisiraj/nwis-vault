---
tags: [concept]
---

# Outbox Pattern

> Up: [[Concepts MOC]]

## The idea

Sometimes you must tell another system something (send a push, an SMS, a mail) but the call can fail or the process can restart. The outbox pattern separates *deciding* to send from *actually sending*. First, write a row that records the intent into a database table, ideally in the same transaction as the business change. Later, a separate worker reads pending rows, sends them, and marks each one done.

The table acts as a queue. Because it is in the database, it survives restarts, you can inspect it with SQL, and a failed send stays pending for the next attempt. You get this without running a message broker.

The trade-off is delivery semantics. If the send succeeds but marking the row fails, the next run sends again. That is **at-least-once** delivery, not exactly-once, so the receiver ideally tolerates duplicates (see [[Idempotency]]).

## In this codebase

**Status: partly present. The word "outbox" appears nowhere in the code (grep of backend, jobs, fake ESB found zero hits), but a table-as-queue with a polling worker does exist.** It is the shape of the pattern, not a textbook implementation.

- **Producer half.** The backend inserts a pending row into `INOUT.INOUT_WBMPLUS_MST` (`tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/repository/inoutads/InoutWbMPlusMstRepository.java:55`, an `insert ... values(...)` that includes `STATUS`). I did not verify that this insert shares a transaction with a business change, so the "atomic with the business write" guarantee is unconfirmed.
- **Worker half.** The `MplusSendMes` job (`tgmdp_jobs/.../job/impl/mdp/MplusSendMes.java:121-151`) drains it. The query lives in `tgmdp_backend/sharedModules/.../repository/inout/InoutWbmplusMstRepository.java`:

```sql
SELECT BMP_ID, ACC_MPLUS_KEY, SEND_MES, PUSH_TIME FROM INOUT.INOUT_WBMPLUS_MST
WHERE STATUS = '0' AND MPLUS_TIME >= (CURRENT TIMESTAMP - 30 MINUTES)
AND (PUSH_TIME IS NULL OR PUSH_TIME <= 3)
```

- **Mark done / retry.** On an M+ success the job calls `updateWbMplusMst(campaignId, "1", pushTime+1, bmpId)` (line 139). On failure it writes `"0"` and still increments `PUSH_TIME` (line 150). So the WHERE clause is the retry policy: at most ~4 attempts, and only inside a 30-minute window. A failure also queues an alert mail by inserting into the mail table (`sendAlertMail`, line 80).
- **Why a table, not a broker.** The company's DB2 is already shared by the legacy TGMDP and the new stack, so any app that can reach DB2 can enqueue. The vault notes say there was no broker; I did not find a broker dependency either.
- **Gap.** If M+ accepts the message and the `UPDATE` then fails, the row stays `'0'` and is re-sent: at-least-once. See [[Job Reliability - Idempotency, Retries, Locking and Failure]].

Related queue-by-status tables exist for mail and SMS (the job inserts mail rows with status `"A"`), but those are drained by other jobs; see [[How Jobs Read and Write Data]] and [[Integration Landscape]].

## Check yourself

Which line of `MplusSendMes` would you have to look at to see why a message that failed five times is never retried again? (Hint: it is not in Java.)

## Related

- [[Idempotency]] · [[Scheduled Jobs and Batch Processing]] · [[Database Transactions]]
- [[Message Integration and ESB]] · [[Concepts MOC]]
