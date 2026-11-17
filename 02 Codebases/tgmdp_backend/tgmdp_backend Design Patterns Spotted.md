---
tags: [codebase, pattern, tgmdp-backend]
---
# tgmdp_backend Design Patterns Spotted

Up: [[tgmdp_backend]] · [[System Overview]]

A catalogue of design patterns you can find in this repo, each with **a concrete file to open** and **why the pattern earns its place here**. Patterns are not goals in themselves. Each one below solves a problem this backend actually has.

All paths are relative to `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/` unless they start with `sharedModules/` or `build.gradle`.

## Architectural patterns

### Strangler Fig
**Where:** the whole repo. Look at legacy URL prefixes kept on purpose: `controller/acptads/ApiH2AcptController.java:22` (`/ACPT_SERVICE/api`), `controller/mrmads/MrmAdsInboundESBController.java:14` (`/MRM_ADS/api/NWIS`), `config/CxfConfig.java` (`/AMSService`, `/NwisMdpSoapHttpService`), and parameter renaming in `filter/LegacyQueryParamFilter.java`.
**Why:** legacy WARs are replaced one module at a time. Callers (ESB, old apps) keep their URLs and parameter names while the implementation behind them is new. Parity comments (`Legacy LoginModel.java:93-118`) prove each rule was carried over.

### Layered Architecture / Service Layer
**Where:** `controller/tgmdpweb/projectAcceptance/PoCategoryController.java` → `service/tgmdpweb/projectAcceptance/PoCategoryServiceImpl.java` → `repository/tgmdpweb/projectAcceptance/PoCategoryBrowseRepositoryImpl.java`.
**Why:** HTTP concerns, business rules and SQL change for different reasons. The service is also the transaction boundary (`@Transactional` on `update`, `:197`). See [[tgmdp_backend Layering and Packages]].

### MVC (REST flavour)
**Where:** `@RestController` classes are the controller, `dto/*ResponseDto` + Jackson are the view, and services + repositories + entities are the model.
**Why:** Spring MVC's structure. The "view" is JSON rendered by Jackson rather than a JSP. That is the main change from the legacy WebWork/JSP apps.

### Front Controller
**Where:** Spring's `DispatcherServlet` (all `/api/**` requests); the CXF servlet under `cxf.path=/TGMDP/services` (`application.properties:9`) for SOAP. `interceptor/SoapActionRoutingInterceptor.java:25-40` picks the operation from the `SOAPAction` header, but it is a routing *interceptor* inside the CXF chain, not a front controller itself.
**Why:** one entry point per protocol does the routing, so individual handlers don't parse URLs or SOAP envelopes.

## Request pipeline patterns

### Interceptor and Filter Chain
**Where:**
- Servlet filters: `middleware/LogFilter.java` (`@Order(HIGHEST_PRECEDENCE)`), `filter/SystemRouteFilter.java`, `filter/LegacyQueryParamFilter.java`.
- Security filters: `security/CsrfOriginCheckFilter.java`, `security/AuthTokenFilter.java`, ordered in `security/WebSecurityConfig.java:164-168`.
- CXF interceptors: `interceptor/SysIdInInterceptor.java`, `WSS4JInInterceptor`, logging interceptors in `config/CxfConfig.java:131-146`.
- HTTP client interceptor: `config/HttpClientConfig.java` `.requestInterceptor(...)` for ESB REST logging.
- Exception "interceptor": `config/GlobalExceptionHandler.java` (`@RestControllerAdvice`).
**Why:** cross-cutting work (logging, auth, context, CSRF) runs once for every request without each controller calling it. Each link either handles the request or passes it on with `chain.doFilter(...)`.

### Proxy (Spring AOP)
**Where:** every `@Transactional` service (`PoCategoryServiceImpl.java:197`) and every `@PreAuthorize` controller method (`PoCategoryController.java:48`) is called through a Spring-generated proxy that runs the transaction or security check around the real method.
**Why:** declarative behaviour. It also explains a classic trap: calling a `@Transactional` method from **the same class** bypasses the proxy, so no transaction starts.

### Decorator
**Where:** `context/ContextCopyingTaskDecorator.java` wraps every `Runnable` given to `applicationTaskExecutor` (`config/AsyncConfig.java:21`); `ContentCachingRequestWrapper`/`ContentCachingResponseWrapper` in `LogFilter.java:54-55`; `new P6DataSource(hikari)` in `config/DataSourceConfig.java:33`; `ParameterRequestWrapper extends HttpServletRequestWrapper` in `LegacyQueryParamFilter.java:178`.
**Why:** add behaviour (copy context, cache body, log SQL, rename params) without changing the wrapped object or its callers. Same interface, extra responsibility.

## Data access patterns

### Repository Pattern
**Where:** Spring Data interfaces such as `repository/acptads/AcptWorkTypeMstRepository.java`; custom fragments `PoCategoryBrowseRepositoryCustom` + `PoCategoryBrowseRepositoryImpl`; shared ones in `sharedModules/.../repository/`.
**Why:** services ask for data by intent (`browse(criteria, pageable)`) and stay ignorant of JPQL vs native SQL vs JDBC. That freedom matters here because DB2 quirks often force a switch of technique. See [[tgmdp_backend Persistence and DB2]].

### Template Method
**Where:**
- `OncePerRequestFilter`: Spring's `doFilter` is the fixed algorithm, and subclasses fill in `doFilterInternal` (`AuthTokenFilter.java:47`, `CsrfOriginCheckFilter.java:55`, `LogFilter.java:50`).
- `AbstractSoapInterceptor.handleMessage` in `SoapActionRoutingInterceptor.java:17-25`.
- `@MappedSuperclass BaseEntityAcpt` with `@PrePersist onCreate()` / `@PreUpdate onUpdate()` hooks (`sharedModules/.../model/acptads/BaseEntityAcpt.java`).
- The "template + callback" family: `NamedParameterJdbcTemplate`, `TransactionTemplate` (`service/acptads/AcptApiH2Service.java:126`, `:275`).
**Why:** the framework owns the invariant steps (run once per request, open/commit/rollback, acquire/release a connection) and you supply only the varying part.

### Unit of work per record (programmatic transactions)
**Where:** `AcptApiH2Service` runs each batch record in `PROPAGATION_REQUIRES_NEW` (`:100-127`).
**Why:** isolate failures inside a batch. See [[Database Transactions]].

## Object creation patterns

### Builder
**Where:** Lombok `@Builder` DTOs, e.g. `PoCategorySearchResponseDto.builder()...build()` (`PoCategoryServiceImpl.java:139-147`), `OtpVerifyResponseDto.builder()` (`service/tgmdpvendor/TgmdpvendorAuthService.java:596-617`); Spring's `ResponseCookie.from(...).httpOnly(true).secure(...).build()` (`controller/tgmdpvendor/TgmdpvendorAuthController.java:318-327`); `RestClient.Builder` (`HttpClientConfig.java:66`).
**Why:** objects with many optional fields are readable to construct and immutable once built. A 20-argument constructor is error-prone.

### Factory (static factory methods)
**Where:** `config/ResponseHelper.java` (`success(...)`, `error(...)`, `validationError(...)`, `unauthorized(...)`); `ApiResponse.errorMessage(...)`; `AuthException.userNotFound()` / `AuthException.invalidDbToken()` used in `AuthTokenFilter.java:69`; `config/JasyptEncryptorFactory.create(...)`; CXF's `JaxWsProxyFactoryBean` in `service/nwisads/NwisSoapService.java:66-80`.
**Why:** named constructors say what they make, and centralise how a standard object is assembled (status + message + timestamp).

### Singleton
**Where:** every Spring `@Service`, `@Repository`, `@Component` bean (default singleton scope); stateless constants such as `SoapClientUtility.TRUST_ALL` and the static `ROUTE_KEYWORD_MAP` in `config/SysRouteMapping.java:17`.
**Why:** one shared, stateless instance per app is cheap and thread-safe. The consequence is that beans **must not** keep per-request state in fields. Per-request state goes in `ThreadLocal`s like `SystemContext` and MDC instead.

## Integration patterns

### Gateway
**Where:** `service/nwisads/NwisSoapService.java`, `AmsSoapService`, `FtsSoapService`, `client/MetadataSoapClient.java`; REST callers like `service/tgmdpweb/projectAcceptance/AcptExternalCallbackServiceImpl.java`.
**Why:** one class per external system holds endpoint, credentials, timeouts and protocol details. The rest of the app calls a Java method.

### Facade
**Where:** `utils/SoapClientUtility.java` (`configurePort`, `configurePortWithWsSecurity`) hides CXF's `BindingProvider`, `ClientProxy`, `HTTPConduit`, `HTTPClientPolicy` and TLS parameters behind one call. (`ResponseHelper` is a static factory, listed under Factory above, not a facade.)
**Why:** a simple interface to a complicated subsystem, and one place to fix things.

### Adapter
**Where:** `filter/LegacyQueryParamFilter.java` adapts legacy `SITE_ID` / `PAGE_NUMBER` (1-based) parameters to the new `siteId` / `page` (0-based); `jaxb/DataHandlerAdapter.java` (`extends XmlAdapter<byte[], DataHandler>`) adapts binary SOAP attachments; `HttpClientConfig.BufferedClientHttpResponse` adapts a consumed response stream back into a readable `ClientHttpResponse`.
**Why:** make two incompatible interfaces work together without changing either side.

### Anti-Corruption Layer
**Where:** `service/nwisads/NwisAuthorityCheckService.java:51-110` translates the ESB's `NwisAccountAuthorityCheckResponseType` (generated SOAP type) into the app's `NwisAuthorityCheckResponseDto`. Generated `com.taiwanmobile.esb.wsdl.*` classes never leak into controllers.
**Why:** the external model (field names, status codes like `CHECK_STATUS=1/2`) stays at the border. If the ESB contract changes, only the translating service changes.

### Outbox (via a DB table)
**Where:** `TgmdpvendorAuthService.sendOtp` inserts the OTP message into `DB2ADMIN.A_MPLUSVEN_LOG` (`:365-387`) rather than calling the M+ messenger directly. Another process delivers it.
**Why:** the message is written in the same database (and transaction) as the OTP itself, so you never get "OTP saved but message lost" or the reverse.

### Observer (callback listener)
**Where:** `logging/EsbLogEventSender.java` implements CXF's `LogEventSender`. CXF's logging interceptors notify it with a `LogEvent` for each SOAP request/response/fault (`send(LogEvent event)`, `:20`). I found no Spring `ApplicationEvent` / `@EventListener` usage in `src/main/java`.
**Why:** CXF doesn't know about ELK, and the sender doesn't know about CXF internals. They meet through a small listener interface.

## Behavioural patterns

### Strategy
**Where:**
- `PasswordEncoder` → `security/Md5PasswordEncoder.java` plugged in at `WebSecurityConfig.java:45-49`.
- `AuthenticationEntryPoint` → `security/AuthEntryPointJwt.java`.
- `TaskDecorator` → `ContextCopyingTaskDecorator`.
- Per-endpoint WS-Security password callbacks passed into `wssProperties(...)` in `CxfConfig.java:301-310` (`NwisMdpPasswordCallback`, `BtsEsbPasswordCallback`, `SoapInboundPasswordCallback`).
**Why:** the framework defines *what* must happen (encode a password, answer an unauthenticated request, check a SOAP password). The app plugs in *how*, without touching framework code. That is the Open/Closed principle.

**A missed opportunity:** `AuthTokenFilter.java:87-207` chooses an authentication strategy with `if (requestUrl.contains("/tgmdpweb/")) ... else if (...contains("/acptads/"))`. Extracting one `AuthStrategy` per client type, chosen by a token claim, would make each branch testable on its own. Good refactoring exercise.

## Try it yourself

Pick any two patterns above and find a **second** example of each in a different module (`amsads/`, `mrmads/`, `registry/`). Write down the file and line. If you can't find a second Outbox, ask yourself why the team might not have needed one elsewhere.

## Related

- [[tgmdp_backend]] · [[System Overview]] · [[tgmdp_backend Layering and Packages]] · [[tgmdp_backend Request Lifecycle]] · [[tgmdp_backend ESB Integration]]
- [[Strangler Fig]] · [[Layered Architecture]] · [[Service Layer]] · [[MVC]] · [[Front Controller]] · [[Interceptor and Filter Chain]]
- [[Repository Pattern]] · [[Template Method]] · [[Builder]] · [[Factory]] · [[Singleton]] · [[DTO Pattern]]
- [[Gateway]] · [[Facade]] · [[Adapter]] · [[Anti-Corruption Layer]] · [[Observer and Pub-Sub]] · [[Strategy]]
- Proxy · [[Decorator]] · [[Outbox Pattern]]
- [[SOLID]] · [[Dependency Injection]] · [[Inversion of Control]] · [[Database Transactions]]
