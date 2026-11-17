---
tags: [practice]
---
# Unit Testing

Up: [[Best Practices MOC]]

## The idea
A unit test runs one small piece of code in isolation, with its collaborators replaced by mocks or stubs, and checks the result. It should be fast, deterministic and independent of the network or a database. Higher-level tests (integration, E2E) cover the wiring.

## Why it matters
- Fast feedback and a safety net for refactors.
- Tests record intended behaviour, including odd legacy behaviour you must keep.

## In this codebase
- **Backend (JUnit 5, Mockito, AssertJ):** `tgmdp_backend/CLAUDE.md:36-48` states the stack and the `should_<outcome>_when_<condition>` naming. There are about 870 `*Test.java` files under `tgmdp_backend/src/test`. In `build.gradle` the default `test` task uses `excludeTags 'integration', 'esbIntegration'`, and a separate `integrationTest` task (`:117`) runs the DB/ESB ones. Reason: unit tests must run anywhere, including the CI Postgres service, without DB2 or a live ESB.
- **Contract tests pin legacy behaviour:** `.../dto/NwisLegacyRequestContractTest.java:34-58` (`should_readLegacyKeys_when_appLogout`) checks the new DTOs still accept the legacy key names. Parity is the main test target in this project.
- **Front ends (Vitest + Testing Library):** `tgmdp_web_frontend/vitest.config.ts:13-25` uses jsdom, `vitest.setup.ts`, and `globals: true`; tests sit in `__tests__/` or `__test__/` beside the code (e.g. `.../project-acceptance-audit-operation/api/__tests__/additionalApis.endpoint.test.ts`, which asserts the final URL built from `apiVersion + basePath + url`).
- **RN shell (Jest with SWC):** `tgmdp_rn_app/package.json:40` runs `jest --config=swc.jest.config.js --coverage`; `:41` adds a Vitest-based `test:api-contract` suite against dev or staging.
- **Fake ESB as the next tier:** unit tests mock SOAP clients; [[Why a Fake ESB Exists]] covers what sits between them and UAT.
- Gap: backend tests mock `SoapClientUtility`, so a changed call can NPE (`port is null`); `CLAUDE.md:44-47` warns to update the stubs together with the service.

## Check yourself
Why are `integration`-tagged tests excluded from the default `gradle test`, and what runs them?

## Related
- [[Test Coverage Gates]] · [[Feature Flags and Mock Toggles]] · [[Code Review]] · [[Testing Pyramid]] · [[Mocking]] · [[tgmdp_backend Testing and Build]] · [[Best Practices MOC]]
