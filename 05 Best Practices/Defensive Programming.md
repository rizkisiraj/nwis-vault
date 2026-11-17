---
tags: [practice]
---
# Defensive Programming

Up: [[Best Practices MOC]]

## The idea
Assume inputs, dependencies and other code can be wrong: validate at boundaries, default nulls, fail safe, clean up in `finally`, and turn dangerous features off by default. Do it at trust boundaries rather than everywhere, or the code fills with noise.

## Why it matters
- Production data is messier than test data.
- Failures should be contained, not cascade.

## In this codebase
- **Cleanup that always runs (vendor portal):** `tgmdp_vendor_frontend/src/features/auth/store/index.ts:22-29` wraps `authApi.logout()` in `try/catch/finally`; the `finally` always does `set({ username: null, userData: null })`. Why: a user whose logout call fails must still be logged out locally.
- **Secure defaults, relax locally:** `app.ota.auth-enabled:true` is the Java default in `OtaSecurityHelper.java:24` and `WebSecurityConfig.java:28`; only `application-local.properties:22` disables it. `CLAUDE.md:205-212` explains the dev skip flags are authorization bypasses if they leak, so they default to `false`.
- **Boundary hardening:** `tgmdp_backend/.../security/CsrfOriginCheckFilter.java` is wired into `WebSecurityConfig`; `LogFilter.java:38` masks sensitive JSON fields. The fake ESB turns off DTDs when parsing XML: `tgmdp_fake_esb/.../endpoint/MetadataEndpoint.java:35` sets `disallow-doctype-decl` (XXE protection) and `:102-104` enables secure processing.
- **Null safety learned from parity bugs:** `.claude/skills/parity-review/SKILL.md` (~line 393) says a bare `.replace()` or `.trim()` on a nullable JPA projection getter is a finding on sight: legacy returned `""`, the rebuild would 500.
- **Network-call limits:** `tgmdp_web_frontend/src/shared/api/client.ts:47,57-70` sets a 10s `timeout` and an `AbortController` that cancels every in-flight request on 401. In the RN shell, recent commits add a gNB axios timeout and a circuit breaker (`fix/1066-gnb-axios-timeout`).
- **Contrast:** legacy adapters catch everything and return HTTP 200 with an error body; the rebuild fails with 4xx/5xx, so compare user-visible outcomes (SKILL.md rule 13).

## Check yourself
Why is it safer for the OTA auth flag to default to `true` in code and be switched off in a local profile, instead of the reverse?

## Related
- [[Secrets Management]] · [[Validation]] · [[Error Handling]] · [[SOLID]] · [[tgmdp_backend Error Handling and Validation]] · [[Fake ESB Internals]] · [[Best Practices MOC]]
