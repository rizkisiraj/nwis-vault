---
tags: [concept]
---

# Testing Pyramid

Up: [[Concepts MOC]]

## The idea

The testing pyramid says to write many fast unit tests, fewer integration tests and very few slow end-to-end tests. Lower layers are cheap, precise and stable; upper layers are realistic but slow and brittle.

The point is feedback speed and diagnosis. When a unit test fails you know where; when an end-to-end test fails you only know something is wrong somewhere.

Key points:

- **Unit**: one function or class, dependencies replaced by fakes.
- **Integration**: several parts together, such as a service with a real or fake external system.
- **End-to-end**: the whole system, often through the UI.
- **Coverage gates**: a measurable minimum that keeps the base layer healthy.

## In this codebase

The pyramid is real in the modern repos and absent in the legacy.

- **Backend: layers by JUnit tag.** `tgmdp_backend/build.gradle:96-99` makes the default `test` task exclude tags `integration` and `esbIntegration`, because they need a live DB or ESB. Two more tasks include them: `integrationTest` (line ~120, loads `.env`, needs DB2) and `esbIntegrationTest` (line 142-146). So `./gradlew test` is the wide base (roughly 875 files named `*Test*.java`, 4 parallel forks, JaCoCo report after each run, lines 91-114), and the narrow tips are opt-in. There is a middle layer too: the SOAP wire tests, which run a real CXF endpoint in memory with mocked services (`AmsSoapWireTest`, see [[Contract Testing]]). I found no JaCoCo *verification* rule, so the backend has a coverage report but no failing gate.
- **Jobs.** `tgmdp_jobs` has about 47 test files, e.g. `MplusSendMesTest.java`, using H2 (`build.gradle:196`) in place of DB2.
- **Frontends: coverage gates.** `tgmdp-app-ams/vite.config.ts:243-248` and `tgmdp_rn_app/package.json:216-223` both require **100%** lines, functions, branches and statements, and fail the build otherwise. That is a very strict base layer. `tgmdp_web_frontend` and `tgmdp_vendor_frontend` have a `test:coverage` script (`vitest run --coverage`) but I found no threshold in their configs. See [[Test Coverage Gates]], [[Web Console Build Lint and Testing]], [[Vendor Portal Build Lint and Testing]].
- **End-to-end.** `tgmdp_web_frontend/playwright.config.ts:29` points Playwright at `./src/test/integration`, but that directory does not exist in the checkout, so E2E is configured, not populated. Real end-to-end checking here is mostly manual or via the simulator-driven test sheets at the workspace level (`testcases/`).
- **Legacy.** `legacy/TGMDP` has no test source tree; the only `*Test*.java` hits are production classes with "Test" in the name (e.g. `getCTI_TestReportFile.java`). That is why parity is checked by reading code: [[Strangler Fig Migration and Legacy Parity]].

Shape of the pyramid in practice: huge unit base, a notable middle of in-memory SOAP/fake-ESB tests, a thin and mostly opt-in top.

## Why it matters

Knowing the layer a test belongs to keeps the suite fast and tells you what not to test at each level.

## Check yourself

Why does the backend exclude `integration` tests from the default `test` task instead of just letting them fail when no DB is present?

## Related

- [[Unit Testing]] · [[Test Coverage Gates]] · [[Mocking]] · [[Code Review]] · [[Environment Separation]]
- [[Why a Fake ESB Exists]] · [[tgmdp_fake_esb]] · [[tgmdp_backend Testing and Build]] · [[Concepts MOC]]
