---
tags: [practice]
---
# Environment Separation

Up: [[Best Practices MOC]]

## The idea
Run the same code in several isolated environments (local, test, UAT, staging, production), each with its own database, endpoints and credentials, and select the environment by configuration, not by editing code. Nothing in a lower environment should be able to touch production data.

## Why it matters
- Safe testing and rehearsal of releases.
- One build artifact behaves differently only through config.

## In this codebase
- **Backend, Spring profiles:** `tgmdp_backend/src/main/resources/application.properties:6` sets `spring.profiles.active=${SERVER_STAGE:local}`, and there is one overlay per environment: `application-local`, `-test`, `-uat`, `-staging`, `-production.properties`. `.env.sample` documents `SERVER_STAGE=[local|test|uat|staging|production]` and gives each stage its own `*_DB_HOST/PORT/NAME/USER/PASSWORD/URL` key set (names only). `application-production.properties` reads `${PRODUCTION_DB_URL}`. `DataSourceConfig.java:26,41` uses `@Profile("local")` / `@Profile("test")` beans. Logging also varies per profile via `<springProfile>` blocks in `logback-spring.xml:300,329`.
- **CI builds per environment:** `tgmdp_backend/.gitlab-ci.yml` defines `build_testing`, `test_testing`, `deploy_testing`, then `_staging` and `_production` variants, each passing `-Penv=${ENV_NAME}` (`:26`) and `--spring.profiles.active=${ENV_NAME}` (`:83`). Builds are `when: manual` (`:36`) so promotion is a human decision.
- **Dev-only bypasses default off:** `application.properties:195` has `app.ota.auth-enabled=${APP_OTA_AUTH_ENABLED:true}`, while only `application-local.properties:22` turns it `false`. `tgmdp_backend/CLAUDE.md:205-212` warns that `nwis.*.skip-enabled` flags must stay false off local. Secure default, relax per environment.
- **RN shell, two axes:** `tgmdp_rn_app/app.config.js:6` reads `APP_ENV` (default `development`) and loads `configs/app-config.{development,staging,production}.json`; see [[RN Shell Build and Environments]] for `APP_VARIANT`.
- **Web front ends:** `VITE_*` variables from `.env` (gitignored) with a committed `.env.example` (`tgmdp_web_frontend`, `tgmdp_vendor_frontend`); `.env.production` is also gitignored in the web console.
- **Fake ESB:** `tgmdp_fake_esb` stands in for the ESB in local/test ([[Why a Fake ESB Exists]]); `.env.sample` is tiny.
- Legacy: environment differences were baked into packaged configs; not examined here.

## Check yourself
What happens to `app.ota.auth-enabled` if someone forgets to set it on a new server?

## Related
- [[Secrets Management]] · [[Feature Flags and Mock Toggles]] · [[Configuration and Environments]] · [[tgmdp_backend Logging and Configuration]] · [[Best Practices MOC]]
