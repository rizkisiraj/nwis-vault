---
tags: [concept]
---
# Configuration and Environments

Configuration is anything that changes between deployments (database host, API URLs, feature switches, credentials) and should live outside the code. The same build runs in dev, test, staging and production by loading a different config, via property files, environment variables, or build-time flags. Keeping environments distinct prevents tests from touching production data.

## In this codebase
- **Profile selected by one env var.** `tgmdp_backend/src/main/resources/application.properties:6` is `spring.profiles.active=${SERVER_STAGE:local}`. The deploy sets `SERVER_STAGE`; a laptop defaults to `local`. Per-profile files (`application-local|test|staging|uat|production.properties`) then point at different databases with placeholders only, e.g. `application-uat.properties:4` `spring.datasource.url=${UAT_DB_URL}`.
- **Layers, with a default after the colon.** `application.properties:129` `nwis.esb.soap.username=${NWIS_ESB_SOAP_USERNAME:nwisadmin}`, consumed by `@Value("${nwis.esb.soap.username:nwisadmin}")` in `service/nwisads/NwisSoapService.java:55`. Locally a `.env` file is loaded (`spring-dotenv`, `build.gradle:239`; `bootRun` reads it via `loadEnvFile()` at `:374,396`) and `.env.sample` is the committed template. Trap documented in `tgmdp_backend/CLAUDE.md:184-190`: a UAT deploy left the sample placeholders `[nwis username]` as real values and every SOAP call failed auth. A default or placeholder lets the app start but wrong; sometimes failing fast would be better.
- **Secrets are not in the files.** `application.properties:8` imports `configtree:${SECRETS_DIR:/run/secrets/}` (one file per secret, mounted by the platform), and `build.gradle:240` brings `jasypt-spring-boot` so values can be `ENC(...)`. Tests point `SECRETS_DIR` at a throwaway key (`build.gradle:107`). See [[Secrets Management]].
- **Environment-conditional behaviour in code.** `service/tgmdpvendor/TgmdpvendorAuthService.java:63-64,124` allows the fixed-OTP test-account bypass only when the stage is in `FIXED_OTP_ALLOWED_STAGES = Set.of("local", "test")`; `config/GlobalExceptionHandler.java:375` includes stack traces only for `local`/`test`; `DataSourceConfig` has `@Profile("local")` beans (`:24-26`); cookies depend on `COOKIE_SECURE` (`TgmdpvendorAuthController.java:60`). See [[Environment Separation]].
- **Per-system file config inherited from legacy.** The legacy `IniValue` built paths like `iniPath + SYS_ID + "/logs/"` (`legacy/tgmdp_vendor_legacy/src/com/sys/integration/IniValue.java:150-157`) and loaded properties for the current system. The backend keeps the same shape in `utils/IniValueUtility.java:21,164` with the `iniValue/<SYSTEM>/` folders, so legacy-style code that asks "where is the upload dir?" keeps working.
- **tgmdp_jobs** uses `application.yml` with the same `${VAR:default}` style for its two datasources (`tgmdp_jobs/src/main/resources/application.yml:48-61`).

## Why it matters
Many "works on my machine" issues are config differences, and several production incidents here were an unset or defaulted variable rather than a code bug.

## Check yourself
`NWIS_ESB_SOAP_USERNAME` is missing on a server. What value does `NwisSoapService` use, and is that better or worse than failing at startup?

## Related
- [[Environment Separation]]
- [[Secrets Management]]
- [[Feature Flags and Mock Toggles]]
- [[Build Tools and Dependency Management]]
- [[tgmdp_backend Logging and Configuration]]
- [[RN Shell Build and Environments]]
- [[tgmdp_rn_app]]
- [[tgmdp_vendor_frontend]]
- [[Vendor Portal Build Lint and Testing]]
- [[Concepts MOC]]
- [[System Overview]]
