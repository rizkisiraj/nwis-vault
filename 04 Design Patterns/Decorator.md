---
tags: [pattern]
---

# Decorator

> Up: [[Design Patterns MOC]]

## The idea

A decorator wraps an object, exposes the **same interface**, and adds behaviour before or after delegating to the wrapped object. Callers cannot tell the difference, and the original class is not changed. Because decorators can be stacked, you compose behaviours (logging, caching, context copying) instead of building one giant class.

It differs from Proxy mainly in intent: a decorator adds responsibility, a proxy controls access.

## In this codebase

All four backend examples verified in `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/`:

1. **Task decorator.** `context/ContextCopyingTaskDecorator.java:5` `implements TaskDecorator`. `decorate(Runnable)` reads `SystemContext.getSysId()` on the calling thread, then returns a new `Runnable` that sets it, runs the original, and `SystemContext.clear()` in `finally` (`:14-25`). `config/AsyncConfig.java` installs it on the executor bean `applicationTaskExecutor` (`executor.setTaskDecorator(new ContextCopyingTaskDecorator())`), so every `@Async` task inherits the system id. Why: `SystemContext` is a `ThreadLocal`, which does not cross threads on its own.
   **Limit:** it copies only the sysId. The class has no MDC handling, so `trace_id` set by `LogFilter` is lost on async threads (see [[tgmdp_backend Logging and Configuration]]).
2. **Request/response wrappers.** `middleware/LogFilter.java:54-55` wraps with `ContentCachingRequestWrapper` / `ContentCachingResponseWrapper`, so the body can be read for logging after the controller has consumed it.
3. **Parameter wrapper.** `filter/LegacyQueryParamFilter.java:178` `ParameterRequestWrapper extends HttpServletRequestWrapper` returns renamed parameters while delegating everything else.
4. **Data source wrapper.** `config/DataSourceConfig.java:33` `new P6DataSource(createDataSource(...))` wraps the Hikari pool so p6spy can log SQL without changing repositories (`SqlJsonLogger.java` is the logging side).

Number 3 is both a decorator and an [[Adapter]]; the label depends on intent (here: translate names).

**Not a true Decorator (downgraded).** On the web console `withTaskGuard(taskIds, routes)` (`tgmdp_web_frontend/src/features/auth/components/guards/withRouteGuards.tsx:6-11`) nests routes under a pathless `TaskRouteGuard` element. It is composition by wrapping, similar in spirit to a higher-order component, but it does not implement the wrapped thing's interface; call it "wrapper route" rather than Decorator.

**Legacy.** I found no decorators in the legacy monolith; cross-cutting behaviour was done in filters ([[Interceptor and Filter Chain]]).

## Check yourself
What would `LogFilter` fail to do if it passed the original `request` instead of `wrappedRequest` down the chain?

## Related

- [[Interceptor and Filter Chain]] · [[Adapter]] · [[Facade]] · [[Strategy]]
- [[Design Patterns MOC]]
