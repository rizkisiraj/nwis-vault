---
tags: [pattern]
---
# Strangler Fig

Up: [[Design Patterns MOC]]

## The idea
Replace a legacy system gradually: new code takes over one piece at a time behind the same entry points, until the old system can be switched off. Named after a vine that grows around a tree and eventually replaces it.

```
callers -> same URL -> [new impl | old impl]   (per module)
```

## Why it helps
- No big-bang rewrite.
- Parity with legacy is provable module by module.
- Old and new can run side by side, so you can compare them on the same data.

## In this codebase
This is the central migration strategy, and most of the "pattern" is process rather than a class.
- **Same entry points on the new backend.** `tgmdp_backend` publishes legacy-shaped endpoints so old clients keep working: `ApiH2AcptController` is mapped to `/ACPT_SERVICE/api` (the javadoc says it lives there "not `/MRM_ADS/**`"), `MrmAdsInboundESBController` to `/MRM_ADS/api/NWIS`, and `CxfConfig.java:126` does `endpoint.publish("/AMSService")` for the SOAP service. `WebSecurityConfig.java:94-99` opens `/MRM_ADS/api/**` and `/ACPT_SERVICE/api/**` as inbound ESB paths.
- **Adapting old request shapes.** `filter/LegacyQueryParamFilter.java` (`OncePerRequestFilter`) runs only for a fixed `LEGACY_PATHS` set (`shouldNotFilter`, line 103). It accepts legacy `UPPER_SNAKE` parameters like `SITE_ID`, exposes them under camelCase (`siteId`), maps `PAGE_NUMBER` (1-based) to 0-based `page`, and per-path renames such as `VER`->`version`. A camelCase parameter from the caller always wins (`if (!params.containsKey(camel))`, line 123), so new clients are unaffected. This lets the old NWIS2 app talk to the new code without being rewritten. It is also an [[Anti-Corruption Layer]] in miniature.
- **Old and new run together on shared data.** `HOWTORUN-local-side-by-side.md` runs legacy Cordova AMS2 and the new AMS web layer against backends on one machine, and notes both "reach the **same DB2** ... so the two apps read the same rows. That is what makes the comparison meaningful". The RN shell can show two tiles when access mode is `BOTH`.
- **Parity as a repeatable gate.** Each rebuilt module gets a static old-vs-new review. The skill `.claude/skills/parity-review/SKILL.md` covers vendor, ACPT and AMS families and writes reports to `vendor_parity_review/`, `acpt_parity_review/` (and `ams_parity_review/`). Reports such as `acpt_parity_review/TGMDPs012_..._新舊版比對檢查報告.en.md` cite legacy `file:line` vs new `file:line` and go through many review rounds; sibling `*_accepted-deviations.md` files record differences the team chose to keep. The AMS SD docs have a Legacy Parity companion under `app-sd-by-module/AMS/parity/`. Code carries the same evidence: `tgmdp_vendor_frontend/.../TGMDPVENc003/pages/ListPage.tsx:49-57` explains an odd rule by citing `TGMDPVENc003Q1.jsp:44-48,85-88` and warns not to "improve" it.

Not a textbook strangler: there is no single router sending traffic per module. Coexistence is per app (separate legacy and new apps, same database), and cut-over is decided module by module through the access mode and parity sign-off.

## Check yourself
Why does `LegacyQueryParamFilter` restrict itself to a path whitelist instead of rewriting every request?

## Related
- [[Anti-Corruption Layer]] · [[Adapter]] · [[Facade]] · [[Strangler Fig Migration and Legacy Parity]] · [[tgmdp_backend Design Patterns Spotted]] · [[tgmdp_vendor_frontend]] · [[Legacy Systems]] · [[Native Modules and Legacy Coexistence]] · [[Design Patterns MOC]]
