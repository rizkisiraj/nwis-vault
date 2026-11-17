---
tags: [pattern]
---
# Adapter

Up: [[Design Patterns MOC]]

## The idea
An Adapter converts one interface into another that a client expects, so two incompatible parts can work together without changing either.

```
legacy caller: ?SITE_ID=1&PAGE_NUMBER=1
     Adapter
new code:      siteId=1, page=0
```

## Why it helps
- Lets old callers keep working while new code stays clean.
- The mismatch is handled in one named place.

## In this codebase
1. **Request-parameter adapter (clearest example).** `tgmdp_backend/.../filter/LegacyQueryParamFilter.java` extends `OncePerRequestFilter`. It renames legacy UPPER_SNAKE params to camelCase (`SITE_ID` -> `siteId`, javadoc `:21`, `toCamelCase` at `:157`), maps `PAGE_NUMBER` -> `page` (`RENAMED`, `:43`) and shifts 1-based to 0-based (`LEGACY_PAGE_NUMBER`, `:54`; `toZeroBasedPage`, `:137`). Then it forwards a wrapped request: `filterChain.doFilter(changed ? new ParameterRequestWrapper(request, params) : request, response)` (`:133`). Controllers are written once in the new style; old Android/ESB callers keep their URLs. This is the Strangler Fig in code (see [[Strangler Fig]]).
2. **XML adapter.** `jaxb/DataHandlerAdapter.java:18` `extends XmlAdapter<byte[], DataHandler>` so JAXB can turn binary SOAP attachments into `byte[]` and back.
3. **Stream adapter.** `config/HttpClientConfig.java:188` `BufferedClientHttpResponse implements ClientHttpResponse` wraps an already-consumed response with a saved `byte[]` body (`:193`) so logging can read it and the caller still can too (used at `:129`).

**Adapter vs Decorator:** the filter's wrapper (`ParameterRequestWrapper`, `:178`) is also a decorator, because it keeps the same `HttpServletRequest` interface. The difference is intent: here the point is to translate names and numbering, not to add a feature. See [[Decorator]].

**Legacy vs modern.** The legacy monolith had no need for this: callers and server were deployed together. The adapter exists only because the rewrite replaces servers behind unchanged clients.

## Check yourself
Why does the filter pass non-numeric `PAGE_NUMBER` values through unchanged (`toZeroBasedPage` javadoc) instead of failing itself?

## Related
- [[Facade]] · [[Gateway]] · [[Anti-Corruption Layer]] · [[Strangler Fig]] · [[Decorator]] · [[Native Modules and Legacy Coexistence]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
