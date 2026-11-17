---
tags: [concept]
---

# Asynchronous Request-Callback

> Up: [[Concepts MOC]]

## The idea

Some work takes minutes, far longer than anyone should hold an HTTP connection open. Instead of waiting for the answer, the caller sends a request that only means "please start", and gets a quick acknowledgement. The result is delivered later, when the other side calls back an endpoint of its own.

To connect the two halves, both messages carry the same **correlation id**. The receiver of the callback uses it to match the result to the original request. Because callbacks can arrive twice or never, the design usually needs an idempotency guard and some way to notice missing callbacks (see [[Idempotency]]).

## In this codebase

The worked example is the ticket sync: [[Trace - SyncTicketAllData ESB Round Trip]].

1. **Request ("please start").** The Quartz job `SyncTicketAllData` (`tgmdp_jobs/.../job/impl/mdp/SyncTicketAllData.java:59-82`) inserts a sync row with `STATUS '0'` and a fresh `transactionId`, then calls the ESB SOAP operation `syncTicketByDate` with `body.setMdpTransactionId(transactionId)`. The reply only carries a header status; no ticket data comes back on this call.
2. **Callback.** Later the ESB calls the backend's own SOAP endpoint, published at `/MDPService` (`CxfConfig.java:167`). `MdpSoapEndpoint` (`tgmdp_backend/.../service/mdpads/MdpSoapEndpoint.java:68-120`) matches `mdpTransactionId` to the stored row, saves the attached XML to disk, replies `S`, and answers `F` if the id is already executed or not found.
3. **Hand-off to a thread.** After replying, the endpoint calls `mdpSyncService.syncTicketDataAsync(...)`, an `@Async @Transactional` method (`MdpSyncService.java:126`) that runs on the pool defined in `config/AsyncConfig.java` (core 10, max 50, queue 100). The acknowledgement therefore returns before the heavy DB work finishes. (The method also does a `Thread.sleep(5000)` first, line 133; the reason is not stated in code.)
4. **FTS twin.** `FtsSoapEndpoint` handles `fullFileSyncResult` the same way, keyed by the same `mdpTransactionId` field, with the status flip inside a `transactionTemplate`.
5. **Infrastructure consequence.** Because the ESB must reach the backend, the callback path has to be routed. On staging, nginx forwarded only `/api/`, so callback prefixes got a 404 at the edge. See [[Integration Landscape]].

**Missing-callback detection:** I did not find a timeout sweeper that marks old `STATUS '0'` sync rows as failed; a lost callback leaves the row at `'0'`. Treat that as an open gap, and verify before relying on it.

For local development, `tgmdp_fake_esb` plays both halves. Its `MdpEndpoint.syncTicketByDate` (`tgmdp_fake_esb/.../endpoint/MdpEndpoint.java:73-78`) only logs and acks, and `InboundTriggerController` (`controller/trigger/InboundTriggerController.java:219`, `POST /trigger/mdp/putSyncFile`) lets a developer fire an ESB-to-backend callback by hand. See [[Why a Fake ESB Exists]].

## Check yourself

The callback response says `F` for an already-executed id. Is that a business failure or a transport failure, and how would the ESB tell the difference from the HTTP status alone?

## Related

- [[Idempotency]] · [[Outbox Pattern]] · [[WS-Security]]
- [[Message Integration and ESB]] · [[Concepts MOC]]
