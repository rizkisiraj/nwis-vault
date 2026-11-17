---
tags: [concept]
---

# Concurrency and Threads

## The idea

Concurrency means doing multiple things at overlapping times. A thread is an independent flow of execution inside a process. Shared mutable data between threads causes race conditions, fixed with locks, atomic operations or by avoiding sharing. JavaScript uses a single thread with an event loop and async calls instead.

At the database level, concurrent writers are handled by transactions and locking.

## In this codebase

- **Thread pool for async work (backend).** `tgmdp_backend/.../config/AsyncConfig.java:13-23` defines `applicationTaskExecutor`: core 10, max 50, queue 100, with a `ContextCopyingTaskDecorator` that copies the per-thread `SystemContext` sys-id onto the worker thread and clears it afterwards (`context/ContextCopyingTaskDecorator.java`), because thread-locals do not cross threads by themselves. About 14 services use `@Async`, e.g. `MdpSyncService.syncTicketDataAsync` (`service/mdpads/MdpSyncService.java:126`), which lets a SOAP callback return immediately. See [[Asynchronous Request-Callback]].
- **Hand-rolled fan-out (jobs).** `MoniterAppService` (`tgmdp_jobs/.../job/impl/mdp/moniter/MoniterAppService.java:85-86`) creates a `CountDownLatch` and `Executors.newFixedThreadPool(SystemName.length)`, submits one health check per system, then `latch.await(PROCESS_TIMEOUT, ...)` and `executor.shutdown()` in `finally`. Checks run in parallel, the job waits for all or times out.
- **Locks for lazy caches.** `MediatorService` (backend) marks `reload()` and `validateCache()` `synchronized` (lines 50, 68) and keeps `lastLoadedTimestamp` `volatile`. `TgmdpvendorAccountService.sqeNames()` (line ~351-362) is double-checked locking around a lazily loaded map. Both protect shared in-memory state.
- **Lock-free counters.** `RateLimitInterceptor` (`tgmdp_jobs/.../config/RateLimitInterceptor.java:32-52`) keeps a `ConcurrentHashMap` of per-IP windows with an `AtomicInteger`, then a `synchronized (window)` block for the reset. 60 requests per minute per IP, else HTTP 429.
- **Preventing overlap of jobs.** Quartz `@DisallowConcurrentExecution` (112 classes) and `threadCount: 1` in `application.yml:81`. See [[Scheduled Jobs and Batch Processing]].
- **Known weak spot.** Static `synchronized` init methods (`IniValueUtility.init`) and an in-JVM serial-number generator whose own comment says it is *not* safe across two servers (`tgmdp_jobs/.../utility/SerialNumber.java:13-17`): `synchronized` only works inside one JVM.
- **DB-level.** Concurrent writers rely on `@Transactional` and DB2 row locks; I found no optimistic `@Version` or pessimistic lock annotations in backend code (grep returned none). See [[Database Transactions]].
- **Frontend and RN.** JS is single-threaded; "concurrency" there is async/await and the native-vs-JS split, see [[Native Modules and Legacy Coexistence]] and [[Bridge API Request Flow and Auth]].

## Check yourself

`SerialNumber` uses `synchronized`. Two job servers start at once. Why can both hand out the same number?

## Related

- [[Database Transactions]] · [[Logging and Observability]] · [[Observer and Pub-Sub]] · [[Defensive Programming]]
- [[Job Reliability - Idempotency, Retries, Locking and Failure]] · [[AMS Bridge Client]]
- [[Concepts MOC]] · [[System Overview]]
