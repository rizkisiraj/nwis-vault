---
tags: [legacy, legacy-to-modern, moc]
---

# Legacy to Modern MOC

> Why the NWIS/TGMDP stack was rebuilt, what was gained, what was lost. Up: [[System Overview]] · See also [[Legacy Systems]]

Each note compares one legacy design to its modern replacement using the same screen or flow on both sides.

| Note | Legacy | Modern |
|---|---|---|
| [[WebWork Actions vs Spring Controllers]] | WebWork Action + XML routing | Spring `@RestController` + service layer |
| [[Server-rendered JSP vs React SPA]] | JSP + form post | React page + JSON API |
| [[Cordova+jQuery vs RN Shell + React Web Layer]] | Cordova WebView, globals, adapters | RN shell + React web layer + REST |
| [[Strangler Fig Migration and Legacy Parity]] | Running legacy | Module-by-module replacement, parity reports |

Planned companions (linked from other notes, not yet written): [[DBDAO vs Spring Data Repositories]], [[Session ButtonBean vs JWT Authorities]].

## One-paragraph story

A server-rendered Java console (JSP, WebWork, raw JDBC on DB2) and two Cordova apps backed by RPC-style adapters grew over a decade with no tests. The rebuild adds a stateless Spring Boot API, React front ends and an RN shell. It runs beside the old code on the same DB2 and is checked module by module against legacy source.

## Reading order

1. [[Legacy Systems]] for the "before".
2. [[WebWork Actions vs Spring Controllers]], then [[Server-rendered JSP vs React SPA]].
3. [[Cordova+jQuery vs RN Shell + React Web Layer]].
4. [[Strangler Fig Migration and Legacy Parity]] for how it is done safely.

## Related

- [[System Overview]] · [[Legacy Systems]] · [[Legacy TGMDP Monolith]] · [[ams2-app-legacy]] · [[niws2-app-legacy]]
- [[Strangler Fig]] · [[MVC]] · [[Layered Architecture]] · [[Single Page Applications]] · [[Hybrid Mobile Apps]]
