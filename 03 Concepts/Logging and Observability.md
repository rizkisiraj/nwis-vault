---
tags: [concept]
---
# Logging and Observability

Logging records what the program did; observability is the wider ability to understand a running system from the outside using logs, metrics and traces. Use levels (DEBUG, INFO, WARN, ERROR), include identifiers such as a request or job id so lines can be joined, and never log secrets.

Logs are often the only evidence of what a batch job did at 3 a.m.

## In this codebase
- **One request, one `trace_id`, many streams.** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/middleware/LogFilter.java:58-70` generates `UUID.randomUUID()` per request and puts it in the SLF4J MDC (a per-thread map) with `endpoint`, `http_method`, `client_ip`, `user`, then logs an END line with body, `status_code` and latency (`:90-92`); `MDC.clear()` is in `finally` (`:109`). Logs are JSON through `LogstashEncoder` (`src/main/resources/logback-spring.xml:37`, shipped over TCP at `:219`) so Kibana can filter by field.
- **SQL and ESB logged separately.** `config/SqlJsonLogger.java:26-30` (a P6Spy logger) sets `log_type=sql`, `sql_query`, `sql_exec_ms`, `sql_params` on the MDC; named loggers such as `sql` have their own file appender with `additivity="false"` (`logback-spring.xml:306`) so lines are not duplicated into the root log.
- **Do not log credentials.** `LogFilter.java:37-38` masks fields named `password`, `token`, `authorization`, `secret`, `refreshToken`. `tgmdp_jobs/src/main/java/com/taiwanmobile/tgmdpscheduler/logging/LogRedactor.java` does the same for outbound calls (and notes that its `token` is deliberately *not* masked because it is a device token).
- **Where it breaks: async.** MDC is thread-local. `context/ContextCopyingTaskDecorator.java` copies only `SystemContext.sysId` to `@Async` threads, not the MDC, so async ESB syncs lose `trace_id` and cannot be joined to the request in Kibana. A worked example of why "it logged fine locally" is not the same as "traceable in prod".
- **Jobs log to a table too.** `tgmdp_jobs/.../service/impl/JobLogServiceImpl.java:22-33` inserts a `JobLog` row (name, start/end, execute/success/fail counts) per run, so job history is queryable without reading files; `logging/` holds outbound/SOAP loggers.
- **Legacy.** `legacy/tgmdp_vendor_legacy/src/com/sys/utility/logs/LogUtility.java` fans each message out to registered sinks (`FileLog`, `ConsoleLog`; levels `LOG_FILE`, `LOG_CONSOLE`, `LOG_ALL`), and many classes just `System.out.println(...)` (e.g. `DBDAO.java`). Plain text, per server file, no correlation id.

## Why it matters
You cannot fix what you cannot see. The structured fields (`log_type`, `trace_id`) are what turn a flood of lines into a story for one request.

## Check yourself
You see an ESB error line in Kibana without a `trace_id`. Based on `ContextCopyingTaskDecorator`, what is the probable cause?

## Related
- [[Error Handling]]
- [[Secrets Management]]
- [[Configuration and Environments]]
- [[Concurrency and Threads]]
- [[tgmdp_backend Logging and Configuration]]
- [[Job Reliability - Idempotency, Retries, Locking and Failure]]
- [[tgmdp_jobs]]
- [[Bridge Engine and Message Protocol]]
- [[Vendor Portal Feature Trace - TGMDPVENc003 Serial Number Query]]
- [[Concepts MOC]]
- [[System Overview]]
