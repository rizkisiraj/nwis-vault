---
tags: [pattern]
---
# Interceptor and Filter Chain

Up: [[Design Patterns MOC]]

## The idea
Requests pass through an ordered chain of handlers. Each can act (log, authenticate, rewrite) then pass on, or stop the request. Cross-cutting work happens once, outside controllers.

```
request -> LogFilter -> CsrfOriginCheckFilter -> AuthTokenFilter -> Controller
             each: do work; chain.doFilter(req, res)
```

## Why it helps
- Logging, auth and CSRF are written once.
- Order is behaviour: ordering bugs are real bugs.

## In this codebase
**Servlet filter chain (tgmdp_backend).**
- `middleware/LogFilter.java:31-33` is `@Component @Order(Ordered.HIGHEST_PRECEDENCE)`, so it runs first. It wraps the request/response (`:54-55`), generates a `trace_id` into the MDC (`:63`), calls `filterChain.doFilter(wrappedRequest, wrappedResponse)` (`:82`) and logs after.
- `security/WebSecurityConfig.java:164-168` orders the security filters: `addFilterBefore(authenticationJwtTokenFilter(), UsernamePasswordAuthenticationFilter.class)` then `addFilterBefore(csrfOriginCheckFilter, AuthTokenFilter.class)`, so CSRF Origin check runs before JWT auth. The comment says why: it also blocks pre-auth abuse.
- `AuthTokenFilter.java:48-56` passes through early for server-to-server calls (`basicAuthService.tryServerToServerAuthenticate`) and otherwise always ends with `filterChain.doFilter` (`:216`). `CsrfOriginCheckFilter.java:55-90` lets safe methods and header-less non-browser clients (native mobile) straight through, and answers 403 JSON when a state-changing request's Origin is not in the allow-list, without calling the rest of the chain.
- `LegacyQueryParamFilter.java:133` rewrites params then continues the chain ([[Adapter]]).

**Interceptors on other protocols.**
- SOAP (CXF): `config/CxfConfig.java` adds `SysIdInInterceptor`, ESB logging, `WSS4JInInterceptor`, then `SoapActionRoutingInterceptor` to one endpoint's in-chain, in that order (`nwisMdpEndpoint`, around `:133-143`). Order matters: logging is added before WS-Security, so a message that fails the password check was already logged (my reading of the order, not a stated rule).
- Outbound REST: `HttpClientConfig.java:71` `.requestInterceptor(...)` logs each ESB REST call.
- Exceptions: `config/GlobalExceptionHandler.java` is `@RestControllerAdvice`, an after-the-fact interceptor for errors.

**tgmdp_jobs.** `config/WebConfig.java:38-49` registers four Spring MVC interceptors in an explicit order with a comment: logging, security headers, then rate limit before auth ("block high-speed abuse, including unauthenticated, first").

**Frontend.** Axios interceptors: `tgmdp_web_frontend/src/shared/api/client.ts:71` (request) and `:101` (response, auto-logout on 401 at `:128` and aborts other in-flight requests).

**Legacy.** `legacy/TGMDP/TGMDP/WEB-INF/web.xml:4-23` declares `IPFilter` on `/*` and `FrontcontrollerFilter` on `*.action`. `FrontControllerFilter.java:19-157` checks session, loads `ButtonBean`, then calls `chain.doFilter` (`:157`) and runs post-processing after the action returns. Same pattern, but the permission logic lives in a session object ([[FrontControllerFilter and ButtonBean Permissions]]).

## Check yourself
If `LogFilter` ran after `AuthTokenFilter`, what would you lose from the logs for rejected requests?

## Related
- [[Front Controller]] · [[Template Method]] · [[Observer and Pub-Sub]] · [[Decorator]] · [[Separation of Concerns]] · [[tgmdp_backend Request Lifecycle]] · [[tgmdp_backend Logging and Configuration]] · [[Web Console API Layer]] · [[Vendor Portal API Layer]] · [[Design Patterns MOC]]
