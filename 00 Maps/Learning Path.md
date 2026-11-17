---
tags: [map]
---
# Learning Path

A suggested order, from the big picture down to detail. Roughly 8 stages; skip freely.

## Stage 1: The big picture
1. [[System Overview]]
2. [[Client-Server Architecture]] · [[HTTP and REST]] · [[Layered Architecture]]

## Stage 2: One backend, deeply
3. [[tgmdp_backend]]
4. [[tgmdp_backend Request Lifecycle]]: follow one request end to end
5. [[tgmdp_backend Layering and Packages]]
6. [[Dependency Injection]] · [[Service Layer]] · [[Repository Pattern]] · [[DTO Pattern]]
7. [[tgmdp_backend Persistence and DB2]] · [[Database Transactions]] · [[ORM and JPA]]
8. [[tgmdp_backend Security and Auth]] · [[Authentication and Sessions]] · [[Cookies and CSRF]]
9. [[tgmdp_backend Error Handling and Validation]] · [[tgmdp_backend Logging and Configuration]]
10. [[tgmdp_backend Testing and Build]] · [[Unit Testing]]
11. [[tgmdp_backend Design Patterns Spotted]]

## Stage 3: Talking to other systems
12. [[What an ESB Is]] · [[tgmdp_backend ESB Integration]] · [[Integration Landscape]]
13. [[Why a Fake ESB Exists]] · [[Fake ESB Internals]]
14. [[tgmdp_jobs]] · [[How Jobs Are Scheduled and Triggered]] · [[Job Reliability - Idempotency, Retries, Locking and Failure]]
15. [[Trace - SyncTicketAllData ESB Round Trip]]: everything so far in one trace

## Stage 4: Browser frontends
16. [[tgmdp_web_frontend]] · [[Web Console Folder and Feature Structure]] · [[Web Console API Layer]]
17. [[Web Console State and Forms]] · [[Web Console Routing Auth and Permissions]]
18. [[Web Console Feature Trace - APP Login Logout Record Query]]
19. [[tgmdp_vendor_frontend]] → [[Web Frontends Compared]]
20. [[Single Page Applications]] · [[Client State Management]] · [[Feature-Sliced Structure]]

## Stage 5: Mobile
21. [[Mobile Architecture Overview]] · [[Hybrid Mobile Apps]]
22. [[tgmdp_rn_app]] → [[Bridge Engine and Message Protocol]] → [[Bridge API Request Flow and Auth]]
23. [[tgmdp-app-ams]] → [[AMS State Hooks and Testing]] · [[tgmdp_vendor_rn]]

## Stage 6: The legacy
24. [[Legacy Systems]] · [[WebWork Request Flow]] · [[Front Controller]] · [[MVC]]
25. [[Legacy Data Access with DBDAO]] · [[Legacy Cordova App Anatomy]]

## Stage 7: Old vs new
26. [[Legacy to Modern MOC]] · [[WebWork Actions vs Spring Controllers]] · [[Strangler Fig Migration and Legacy Parity]]

## Stage 8: Practices
27. [[Best Practices MOC]]: [[SOLID]], [[Separation of Concerns]], [[Test Coverage Gates]], [[Linting and Formatting]]

Back to [[Home]].
