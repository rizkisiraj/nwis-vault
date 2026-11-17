---
tags: [concept]
---
# Inversion of Control

Normally your code calls the library. With inversion of control (IoC), a framework calls your code. You write handlers, beans or jobs and register them; the framework decides when to create and invoke them. Dependency injection is one form of IoC (the container creates your objects); scheduling callbacks, servlet filters and web request handlers are others.

The consequence is that control flow is not visible in your own code, so you must learn the framework's lifecycle (startup, request, shutdown). If you cannot find a caller, the framework is the caller.

## In this codebase
- **Spring calls your controller.** In `tgmdp_backend`, nobody calls `PoCategoryController` methods directly. Annotations (`@RestController`, `@PreAuthorize("hasAuthority('TASK_TGMDPs011:browse') ...")` at `controller/tgmdpweb/projectAcceptance/PoCategoryController.java:48-68`) are declarations; the framework routes the URL, runs the security check, binds JSON and calls the method. Legacy WebWork did the same thing from XML: `legacy/TGMDP/TGMDP/WEB-INF/classes/xwork.xml:42-52` maps action names like `Login` to `com.prj.login.Login` and a method name.
- **Filters are called, not invoked.** `security/WebSecurityConfig.java:164-168` registers `AuthTokenFilter` and `CsrfOriginCheckFilter` with `addFilterBefore(...)`. You never call them; every request passes through them in the order you declared.
- **Exception handling inverted.** `config/GlobalExceptionHandler.java` is a `@RestControllerAdvice`: code deep in a service just `throw`s, and the framework finds a matching `@ExceptionHandler` method (about 25) to build the HTTP reply.
- **The scheduler owns the job lifecycle.** In `tgmdp_jobs`, Quartz decides when a job runs; the job class does not start itself. `config/QuartzJobFactory.java` is the hook where Quartz asks "give me a job instance" and the team lets Spring wire it (see [[Dependency Injection]]). Job rows live in the DB, not in code, so what runs is configuration.
- **Counter-example, legacy.** `TGMDPVENc002.java` (in `legacy/tgmdp_vendor_legacy/src/com/prj/actions/tgmdpvenc/`) implements `Action` and `ModelDriven`; WebWork calls `begin()`/`execute()`, but inside the method the code drives everything itself (open DAO, build models, close). Framework-in-control at the edge, hand-written control inside.

## Why it matters
It explains why code with no visible caller still runs, and why moving an annotation or a bean can change behaviour without any call site changing.

## Check yourself
Open `GlobalExceptionHandler`: which line of controller code ever calls `handleNotFound`? (None. What does that tell you?)

## Related
- [[Dependency Injection]]
- [[Front Controller]]
- [[Template Method]]
- [[Observer and Pub-Sub]]
- [[Interceptor and Filter Chain]]
- [[How Jobs Are Scheduled and Triggered]]
- [[tgmdp_jobs]]
- [[WebWork Actions vs Spring Controllers]]
- [[Bridge API Request Flow and Auth]]
- [[Concepts MOC]]
- [[System Overview]]
