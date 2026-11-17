---
tags: [practice]
---
# SOLID

Up: [[Best Practices MOC]]

## The idea
Five design guidelines: **S**ingle responsibility (one reason to change), **O**pen/closed (extend without editing), **L**iskov substitution (subtypes honour the base contract), **I**nterface segregation (small focused interfaces), **D**ependency inversion (depend on abstractions, not concrete classes).

```
class OrderService {            // S: only order rules
  OrderService(Repo r) {...}      // D: gets repo injected, tests pass a mock
}
```

## Why it matters
- Code stays easy to change and test.
- Most useful as a smell detector, not a checklist. Strict SOLID in every file is not the goal.

## In this codebase
- **S, enforced by tooling (RN shell):** `tgmdp_rn_app/eslint.config.js:103-109` caps `max-lines-per-function` at 80, `complexity` at 10, `max-depth` at 3, `max-params` at 8 and `max-lines` at 250. A function that does too much simply fails lint. The team made "single responsibility" measurable because humans reviewing 250-line files miss it.
- **S, in the web console:** `tgmdp_web_frontend/src/features/document-download-area/` splits into `api/`, `pages/`, `types/` and `routes.tsx`. The page renders, the `api/` layer talks HTTP, and neither knows the other's internals.
- **D / O, backend:** controllers and services get collaborators through Spring injection (e.g. `WebSecurityConfig` autowires `AuthEntryPointJwt` and `CsrfOriginCheckFilter`, `tgmdp_backend/.../security/WebSecurityConfig.java:20-26`). Tests swap them for Mockito mocks, which is why the suite is mostly pure unit tests.
- **O, front end:** `tgmdp_web_frontend/src/features/document-download-area/api/index.ts:4-11` exports either the real or the mock implementation behind one shape, so callers do not change when you swap implementations.
- **Counter-example:** `AuthTokenFilter` (`tgmdp_backend/.../security/AuthTokenFilter.java:54-82`) is one method that mixes server-to-server auth, JWT validation, a DB token check and a `requestUrl.contains("/tgmdpweb/")` branch. Adding a new client family means editing it, which breaks Open/Closed.
- **Legacy vs modern:** the legacy WebWork action does request parsing, SQL and rendering in one class (see [[WebWork Actions vs Spring Controllers]]); the rebuild splits those into controller, service and repository.
- Note: the older vault claim that the backend uses a clean Strategy pattern is weak in source; a grep for `Strategy` finds only HTTP client config classes. See [[tgmdp_backend Design Patterns Spotted]] before relying on it.

## Check yourself
Where would you add a fourth kind of caller to `AuthTokenFilter` today, and what would you extract so the next one needs no edit?

## Related
- [[Separation of Concerns]] · [[Strategy]] · [[Service Layer]] · [[Dependency Injection]] · [[Inversion of Control]] · [[Best Practices MOC]]
