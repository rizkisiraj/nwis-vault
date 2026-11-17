---
tags: [pattern]
---
# Singleton

Up: [[Design Patterns MOC]]

## The idea
Only one instance of a class exists and everyone shares it. Old style: a private constructor plus a static `getInstance()`. Modern style: the container or module system guarantees it.

```
// JS: module evaluated once
export const apiClient = axios.create({...})
// Spring: @Service beans default to singleton scope
```

## Why it helps
- Cheap, shared, stateless helpers.
- Rule: no per-request state in fields (use ThreadLocal/MDC or arguments).

## In this codebase
1. **Spring beans (default).** Every `@Service`, `@Repository`, `@Component`, e.g. `PoCategoryServiceImpl` and `LogFilter`, is created once per application context. Hence per-request data lives in `ThreadLocal`s: `context/SystemContext.java:5` holds `SYS_ID` in a `ThreadLocal<String>`; `EsbLogEventSender.java` keeps `outboundStart`/`inboundStart` thread locals. Putting such data in a bean field would be a cross-request bug.
2. **Static constant tables.** `config/SysRouteMapping.java:17` `ROUTE_KEYWORD_MAP` is a `static final LinkedHashMap` filled in a static block ("FIRST MATCH WIN" order) used to map URL keywords to system ids.
3. **Hand-rolled singleton in legacy.** `legacy/TGMDP/src/com/sys/presentation/TaskMediator.java:8-26`:
```java
private static TaskMediator ourInstance;
private TaskMediator() { ... }
public synchronized static TaskMediator getInstance() {
  if (ourInstance==null){ ourInstance=new TaskMediator(); }
  return ourInstance;
}
```
`FrontControllerFilter` calls `TaskMediator.getInstance()` on every request to look up task names. Why a singleton: the task table is loaded once and shared. The cost: global state, `synchronized` on every call, hard to test. Spring beans replaced this style.
4. **Frontend module singletons.** `tgmdp_web_frontend/src/shared/api/client.ts:44` `const apiClient = axios.create({ baseURL, timeout: 10000, withCredentials: true, ... })`, exported default at `:228`; every feature imports the same instance, so interceptors (`:71`, `:101`) apply everywhere. `tgmdp_vendor_frontend/src/lib/client.ts:44` repeats the idea. In AMS, the module-level `subscribed` flag (`api.ts:457`) makes the bridge subscription idempotent.

## Check yourself
Why is MD5 `MessageDigest.getInstance("MD5")` in `Md5PasswordEncoder` NOT a singleton even though it says `getInstance`?

## Related
- [[Factory]] · [[Store Pattern]] · [[SOLID]] · [[Dependency Injection]] · [[Inversion of Control]] · [[Web Console API Layer]] · [[Vendor Portal API Layer]] · [[Design Patterns MOC]]
