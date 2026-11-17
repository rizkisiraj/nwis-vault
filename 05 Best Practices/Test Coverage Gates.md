---
tags: [practice]
---
# Test Coverage Gates

Up: [[Best Practices MOC]]

## The idea
A coverage gate fails the build when tested lines, branches or functions drop below a threshold. Coverage shows what tests never ran; it does not prove the tests are good. A gate works best as a ratchet on code that matters.

## Why it matters
- Stops silent erosion of tests.
- A 100% gate also pushes you to exclude code deliberately and say why.

## In this codebase
Practice is uneven across repos.
- **Strictest, AMS app:** `tgmdp-app-ams/vite.config.ts:243-248` sets thresholds of 100 for lines, functions, branches and statements. The `exclude` list above it (`:159-202`) is the trade-off: UI pages (`src/pages/**`, comment "deep branch coverage requires E2E tooling"), native bridge interop (`src/utils/bridge/**`, untestable in jsdom) and barrel `index.ts` files are excluded, so the 100% applies to logic, hooks and utils.
- **RN shell, also 100%:** `tgmdp_rn_app/package.json:216-223` has `coverageThreshold.global` at 100 for all four metrics. `.gitlab-ci.yml:113` scrapes the `All files` percentage into the pipeline. The `npm run husky` script includes `npm run test`, so the gate runs pre-commit and in CI.
- **Web console, report only:** `tgmdp_web_frontend/vitest.config.ts:13-20` enables v8 coverage and excludes `features/**/api/**`, `types/**` and `routes.tsx`, but sets no `thresholds`. Its `.gitlab-ci.yml` only includes `.autoqa/analyze.gitlab-ci.yml`, a read-only AutoQA test-proposal job.
- **Backend, report only:** `tgmdp_backend/build.gradle` applies `jacoco` and `test` is `finalizedBy jacocoTestReport` (`:114`); no `jacocoTestCoverageVerification` or minimum exists. `sonarqube` is configured (`:360`), so thresholds would live in Sonar. CI (`.gitlab-ci.yml:38-48`) runs `gradle test` per environment and publishes JUnit XML.
- **Vendor portal, none:** `package.json:18` has `test:coverage`, but `@vitest/coverage-v8` is not installed in `node_modules/@vitest` and the repo has no CI file.

## Check yourself
Why would a team exclude `src/pages/**` from a 100% gate instead of lowering the number?

## Related
- [[Unit Testing]] · [[Linting and Formatting]] · [[Code Review]] · [[Vendor Portal Build Lint and Testing]] · [[RN Shell Build and Environments]] · [[Best Practices MOC]]
