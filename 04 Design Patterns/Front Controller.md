---
tags: [pattern]
---
# Front Controller

Up: [[Design Patterns MOC]]

## The idea
One entry point receives all requests of a kind and routes them to handlers, so handlers don't parse URLs or envelopes themselves.

```
all /api/** -> DispatcherServlet -> @Controller method
```

## Why it helps
- Central place for routing and shared setup.
- Handlers stay small.

## In this codebase
**Modern (framework-provided, no project code).** In `tgmdp_backend`, Spring MVC's `DispatcherServlet` is the front controller: it matches the URL to a `@RequestMapping` such as `PoCategoryController`'s `@RequestMapping("/api/v1/tgmdpweb/project-acceptance/po-category")` (`:40`). You will not find it in source, only its configuration. SOAP has a second front door: Apache CXF's servlet mounted at `cxf.path=/TGMDP/services` (`src/main/resources/application.properties:9`), where `CxfConfig` publishes endpoints such as `/NwisMdpSoapHttpService` and `/AMSService`. In the fake ESB, `tgmdp_fake_esb/.../config/SoapWebServiceConfig.java:103-108` registers Spring-WS's `MessageDispatcherServlet` on `/ws/*`, which routes each SOAP message to an `@Endpoint`.

**Legacy (WebWork), with a naming trap.**
- The real front controller is the servlet `webwork`, mapped to `*.action` in `legacy/TGMDP/TGMDP/WEB-INF/web.xml:24-27, 213-216` and implemented by the project's own `com.sys.servlet.ServletDispatcher` (92 lines, `legacy/TGMDP/src/com/sys/servlet/ServletDispatcher.java:18`). It maps `/TGMDPs011/browse.action` to the action class and method declared in `src/WW_XML/TGMDPs.xml:329-341`.
- `com.sys.filter.FrontControllerFilter` (`legacy/TGMDP/src/com/sys/filter/FrontControllerFilter.java:12`, mapped to `*.action` before the servlet) is **named** "front controller" but is really an interceptor: it checks the session, loads the `ButtonBean` permission object, sets `taskname` and calls `chain.doFilter` (`:157`). Routing is not its job. Treat it as a filter-chain link; see [[Interceptor and Filter Chain]] and [[FrontControllerFilter and ButtonBean Permissions]].

**Not a front controller (earlier claim downgraded).** `interceptor/SoapActionRoutingInterceptor.java:25-40` does not dispatch to handlers. It only fills in the missing `BindingOperationInfo` by matching the `SOAPAction` header against the WSDL's operations, as a repair step inside CXF's own dispatcher.

**Why it matters.** The legacy route table was XML (`TGMDPs.xml`) plus a `.action` URL convention (`/<Task>/<method>.action`); the modern one is annotations on controllers plus a versioned REST path. Same idea, different place for the map.

## Check yourself
Why would adding authentication to the legacy system most naturally go in a filter in front of `ServletDispatcher` rather than inside each action?

## Related
- [[MVC]] · [[Interceptor and Filter Chain]] · [[Gateway]] · [[WebWork Request Flow]] · [[WebWork Actions vs Spring Controllers]] · [[tgmdp_backend Request Lifecycle]] · [[Fake ESB Internals]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
