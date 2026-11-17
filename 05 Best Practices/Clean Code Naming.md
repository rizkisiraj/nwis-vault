---
tags: [practice]
---
# Clean Code Naming

Up: [[Best Practices MOC]]

## The idea
Names should say what a thing is or does, at the right level of detail, so the reader needn't open the body. Conventions (camelCase, test names with a pattern) matter less than being consistent and searchable.

## Why it matters
- Code is read far more than written.
- A good name replaces a comment and survives refactors better.

## In this codebase
- **Test naming rule, written down:** `tgmdp_backend/CLAUDE.md:40` mandates `should_<outcome>_when_<condition>`. Real example: `should_readLegacyItemKeys_when_saveCheckItem` in `tgmdp_backend/src/test/java/com/taiwanmobile/boilerplate/dto/NwisLegacyRequestContractTest.java:34`. A failing test name reads like a sentence.
- **Lint-enforced camelCase on the web console:** `tgmdp_web_frontend/eslint.config.js:147` sets `camelcase: "error"`, alongside `eqeqeq` (`:124`), `prefer-const` (`:139`) and `no-var` (`:144`).
- **Names that map to the legacy system:** vendor features are `TGMDPVENa001` ... `TGMDPVENc004` (`tgmdp_vendor_frontend/src/features/`); the web console uses `TGMDPs012` style codes in branch names (`fix/TGMDPs012-legacy-parity`). The code is the join key between a ticket, a legacy JSP and a new folder. This trades readability for traceability on purpose.
- **Backend URL and package names** follow the upstream system: `/api/v1/nwis`, `/api/v1/ftsads`, `/api/v1/tgmdpweb/...` (`tgmdp_backend/.../controller/`). Package names such as `nwisads`, `amsads`, `oemsads` mirror legacy ADS adapter names.
- **Legacy field names kept on purpose** (e.g. `NOW_PLACE`): see [[Vendor Portal Feature Trace - TGMDPVENc003 Serial Number Query]]. Parity reviews compare against legacy, so renaming would hide diffs.
- **Credential-looking names are avoided in RN:** `tgmdp_rn_app/eslint.config.js:148-149` bans credential-named enum members and points to `SecureStorageKey.BIOMETRIC_REAUTH`, because scanners flag the literal.

## Check yourself
When is it right to keep an ugly legacy field name, and when should you map it to camelCase at the boundary?

## Related
- [[Linting and Formatting]] · [[Code Review]] · [[SOLID]] · [[Best Practices MOC]]
