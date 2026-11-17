---
tags: [practice]
---
# Secrets Management

Up: [[Best Practices MOC]]

## The idea
Passwords, tokens and keys must not live in source control, bundles or logs. Keep them in the environment or a secret store, encrypt at rest where you must commit config, mask them in logs, and scan for leaks before they reach the repo.

## Why it matters
- Git history is forever; a leaked key must be rotated.
- Client bundles are public: a `VITE_*` value is visible to every user.

## In this codebase
- **Env files ignored, templates committed:** `tgmdp_backend/.gitignore:24-26` ignores `.env` and `.env.*`; `.env.sample` lists key names with placeholder text. `tgmdp_jobs/.env.example` and `tgmdp_web_frontend/.env.example` do the same. Keys such as `JWT_SECRET` and `INTERNAL_TOKEN` appear only as names. The backend properties reference them as `${JWT_SECRET}` (`application.properties:31`).
- **Jasypt-encrypted properties:** `application.properties:7-8` imports `configtree:${SECRETS_DIR:/run/secrets/}`, a directory holding the master key, one file per property. `build.gradle:160-166` has a `jasyptEncrypt` task that reads plaintext from stdin so it never lands in a shell history or Gradle log. `.gitignore:67-68` excludes `/secrets/` ("never committed, never baked into an image"); tests use a test-only key via `SECRETS_DIR` (`build.gradle:106-107`).
- **Pre-commit scan:** `tgmdp_backend/.githooks/pre-commit:32` scans the staged diff for `(password|secret|token|jwt)[:=] <8+ chars>`. `CLAUDE.md:48-55` admits it is a heuristic with false positives and says to rename variables rather than use `--no-verify`.
- **Lint as secret scanner (RN):** `tgmdp_rn_app/eslint.config.js:120-149` bans string literals assigned to credential-named properties, variables, JSX props and enum members; `npm run lint:credentials` (`scripts/scan-hardcoded-credentials.ts`) runs it in CI. Tests read fixtures from `__tests__/support/credentials`.
- **Log masking:** `tgmdp_backend/.../middleware/LogFilter.java:38` masks `password`, `token`, `authorization`, `secret`, `refreshToken` in logged bodies.
- **Frontends:** `VITE_*` is inlined into the bundle, so only non-secret values (API base URL, flags) belong there.
- Not verified: RN signing keystores (see [[RN Shell Build and Environments]]).

## Check yourself
Why does the Jasypt master key sit outside the repo while the encrypted values sit inside it?

## Related
- [[Environment Separation]] · [[Defensive Programming]] · [[WS-Security]] · [[tgmdp_backend Logging and Configuration]] · [[Best Practices MOC]]
