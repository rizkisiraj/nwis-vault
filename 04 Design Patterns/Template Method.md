---
tags: [pattern]
---
# Template Method

Up: [[Design Patterns MOC]]

## The idea
A base class fixes the skeleton of an algorithm and leaves selected steps for subclasses (or callbacks) to fill in. The framework owns the invariant steps.

```
abstract class JobExecuter {
  executeInternal() { log start; pick mode; doWork() or run(); log result; send mail }
  abstract doWork();   // subclass fills this
}
```

## Why it helps
- Invariant steps (logging, error capture, open/commit/rollback) can't be forgotten.
- Subclasses stay tiny.

## In this codebase
1. **Scheduled jobs (strongest example).** `tgmdp_jobs/src/main/java/com/taiwanmobile/tgmdpscheduler/job/JobExecuter.java:24` is `abstract class JobExecuter extends QuartzJobBean`. The template is `executeInternal` (`:50`): set MDC `trace_id`, resolve the system id from the package name, refresh settings, write an execute-log row, branch on trigger type (once / event / normal, `:62-78`), catch any exception into an `ERROR` status, update the log and send the completion mail. The hooks are `protected abstract void doWork()` (`:37`) and `run(Map para)` (`:39`). `job/impl/bts/MakeCSV.java:23` `extends JobExecuter` and implements `doWork()` (`:31`) and `run` (`:37`) as thin delegates. Why: every job needs identical audit logging and error handling; with the template, a job author cannot skip it.
2. **Servlet filters.** `AuthTokenFilter.java:32,48`, `CsrfOriginCheckFilter.java:36,55` and `middleware/LogFilter.java:33,50` extend Spring's `OncePerRequestFilter` and only implement `doFilterInternal`; the base class guarantees once-per-request execution.
3. **Entity lifecycle hooks.** `sharedModules/.../model/acptads/BaseEntityAcpt.java:14-17` is a `@MappedSuperclass` with `@PrePersist` (`:31`) and `@PreUpdate` (`:37`) callbacks that fill audit columns for every subclass.
4. **Template + callback.** `TransactionTemplate.execute(status -> ...)` (`service/acptads/AcptApiH2Service.java:126-127, 275`) owns begin/commit/rollback; you pass only the body. `NamedParameterJdbcTemplate` (`PoCategoryBrowseRepositoryImpl.java:18`) does the same for connections and result sets.

**Legacy vs modern.** Legacy WebWork actions each repeated open/commit/rollback/close by hand (`TGMDPs011.java:86-123`: `dao.open()` ... `dao.rollback()` ... `dao.close()`), so there was no template. The modern code moved those invariant steps into framework templates.

Template Method uses inheritance; [[Strategy]] reaches a similar goal with composition.

## Check yourself
In `JobExecuter`, which exceptions can a subclass's `doWork()` throw without stopping the next scheduled run, and why?

## Related
- [[Strategy]] · [[Interceptor and Filter Chain]] · [[SOLID]] · [[Scheduled Jobs and Batch Processing]] · [[tgmdp_jobs]] · [[How Jobs Are Scheduled and Triggered]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
