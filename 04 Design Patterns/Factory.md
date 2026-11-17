---
tags: [pattern]
---
# Factory

Up: [[Design Patterns MOC]]

## The idea
Move object creation into named functions or classes so callers say what they want, not how to assemble it. Static factory methods are the common form; a Factory class picks the concrete type.

```
ResponseHelper.error("msg")      // status + message set for you
AuthException.userNotFound()
```

## Why it helps
- Names explain intent (`invalidDbToken()`).
- Standard objects are assembled the same way everywhere.
- Callers depend on a type, not on a concrete class.

## In this codebase
1. **Static factory methods.** `tgmdp_backend/.../config/ResponseHelper.java`: `success(T)` (`:15`), paged `success(List, Page)` (`:20, :31`, converts the 0-based page to 1-based), `error(String)` (`:46`), `error(HttpStatus, code, msg)` (`:56`), `validationError(...)` (`:64`). About 150 controller files call `ResponseHelper.`, so none type "Operation successful" themselves. `exception/AuthException.java:33,40` offers `userNotFound()` and `invalidDbToken()`; `AuthTokenFilter.java:69` throws `AuthException.invalidDbToken()` and `:102,:113` compare against the constants.
2. **Factory for a configured object.** `config/JasyptEncryptorFactory.java:13,34` `public static StringEncryptor create(String password)` fixes algorithm (`PBEWITHHMACSHA512ANDAES_256`), iterations, salt/IV generators in one place, so all callers get identical encryption settings.
3. **Library factory beans.** `service/nwisads/NwisSoapService.java:67-80` builds an ESB client with CXF's `JaxWsProxyFactoryBean` (set SEI class, force SOAP 1.2 because the WSDL uses `soap12:binding`, set address), then `soap.configurePort(...)`.
4. **Legacy reflective factory.** `legacy/TGMDP/src/com/sys/integration/DAOFactory.java:5-12`: `getDAOFactory("DB")` does `Class.forName("com.sys.integration." + name + "DAOFactory").newInstance()`; `DBDAOFactory.creatDAO("DBDAO")` does the same for the DAO. Every action starts with `DAOFactory.getDAOFactory("DB")` then `creatDAO("DBDAO")` (e.g. `TGMDPs011.java:61-62`; 722 call sites in `legacy/TGMDP/src`). In practice there is only one DAO type, so the factory adds indirection (string-typed, errors swallowed with `System.out.print`) more than flexibility. Spring's dependency injection replaced it.
5. **Fake ESB.** `tgmdp_fake_esb/.../config/SoapWebServiceConfig.java:55-80`: `createWebServiceMessage()` returns a SOAP 1.1 or 1.2 message depending on the request `Content-Type` (`application/soap+xml` vs `text/xml`, BTS special case).

## Check yourself
Why would `ResponseHelper.error("x")` be hard to replace with `new ApiResponse<>(...)` calls spread across 50 controllers?

## Related
- [[Builder]] · [[Singleton]] · [[Strategy]] · [[DTO Pattern]] · [[Legacy Data Access with DBDAO]] · [[Dependency Injection]] · [[Bridge Engine and Message Protocol]] · [[Design Patterns MOC]]
