---
tags: [concept]
---

# Scheduled Jobs and Batch Processing

Up: [[Concepts MOC]]

## The idea

Most software reacts to a user request. Batch processing is the opposite: work that runs on a clock or a trigger, with no user waiting. Nightly syncs, report generation, data clean-up and retries of failed calls are typical.

Batch work has a different shape from request work. It touches many rows, runs for minutes, bursts at fixed times, and must still run when nobody is logged in. That is why it usually lives in its own process, so a slow job cannot starve the web server.

Key points:
- **Trigger**: a cron expression, a fixed delay, or an external scheduler decides when a job starts.
- **Idempotency**: running a job twice must not corrupt data, because retries and overlaps happen.
- **Locking**: when several instances run, only one should execute a given job at a time.
- **Failure handling**: record what failed, retry with limits, and make the failure visible instead of silent.

## In this codebase

- **A separate process.** All scheduling lives in `tgmdp_jobs`; the backend has no `@Scheduled` method (grep found none). Jobs are Quartz `QuartzJobBean` subclasses under `tgmdp_jobs/src/main/java/com/taiwanmobile/tgmdpscheduler/job/impl/` (about 30 in `mdp/` alone: `SyncOrgData`, `SendSmsCheck`, `MplusSendMes`, `SyncTicketAllData`, ...).
- **Quartz with a database store.** `config/QuartzConfig.java:23-31` builds the scheduler on its own `quartzDataSource`. `application.yml:69-89` sets `job-store-type: jdbc`, `JobStoreTX` with the DB2 delegate, `tablePrefix: QRTZ_`, `threadCount: 1` and `isClustered: false`. So triggers survive restarts, but there is one worker thread (production profile, line 181; UAT raises it to 20 at line 238) and no cluster lock. Running two instances would double-fire jobs; the config does not defend against that.
- **Triggers are data, not code.** Cron strings are created through an HTTP API: `job/controller/ScheduleController.java` calls `schedulerService.scheduleJob(...)` / `rescheduleJob(...)` (lines 145, 413, 460), and `/triggerJob` (line 190) fires one run on demand, guarded by `isReplay(...)` (line 195) to block rapid repeats.
- **One template for every job.** `job/JobExecuter.java` (`executeInternal`) wraps `doWork()`: sets a `trace_id` in the MDC, logs a start row, picks the mode from the trigger (a "once" trigger, an event trigger with no next fire time which calls `run(dataMap)`, or normal cron), catches any exception, saves status `FINISHED`/`ERROR` and sends a completion mail. This is the Template Method pattern: [[Template Method]].
- **Overlap guard.** `@DisallowConcurrentExecution` is on 112 job classes (e.g. `MplusSendMes.java:25`, `SyncTicketAllData.java:20`) so a slow run is never re-entered by its own next tick.
- **Failure handling.** Jobs usually catch, log, and email an alert instead of rethrowing (`MplusSendMes.sendAlertMail`, line 80). The status of each run is stored by `ExecuteLogService`.
- **A comment worth reading.** `SendSmsCheck.java:61-62` explains the author removed a `@Transactional` from a private self-called method because it never took effect (Spring proxies skip self-calls). `SyncTicketAllData.doSync` has the same shape (`@Transactional private void doSync()`), so assume it is not transactional either.

Legacy vs modern: the legacy monolith ran periodic work inside the web container; here the scheduler is its own deployable.

## Check yourself

With `threadCount: 1`, can `@DisallowConcurrentExecution` ever matter in production? Think about what the annotation protects when the thread count is raised, as in UAT.

## Related

- [[Concurrency and Threads]] · [[Defensive Programming]] · [[Template Method]] · [[Message Integration and ESB]] · [[Configuration and Environments]]
- [[tgmdp_jobs]] · [[How Jobs Are Scheduled and Triggered]] · [[Job Reliability - Idempotency, Retries, Locking and Failure]] · [[tgmdp_backend Logging and Configuration]]
- [[Concepts MOC]]
