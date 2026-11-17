---
tags: [concept]
---

# Internationalization

Up: [[Concepts MOC]]

## The idea

Internationalization (i18n) is designing software so text, dates, numbers and layout can adapt to a language or region. Localization is supplying the actual translations.

Good i18n keeps user-facing strings out of code in message tables keyed by id. Smaller projects sometimes skip the framework and hard-code a single language, which is simpler but costly to change later.

Key points:

- **Message catalogs**: text lives in per-language files.
- **Formatting**: dates, currency and plurals follow locale rules.
- **Encoding**: UTF-8 end to end so Chinese characters survive.
- **Error text**: server messages and client translations must agree.

## In this codebase

**There is no i18n framework.** No `i18next`, `react-intl` or `useTranslation` appears in any frontend `package.json`, and every `index.html` says `<html lang="en">`. The backend has no message-bundle setup (`MessageSource` / `ResourceBundle` hits are incidental). The product is single-language Traditional Chinese, hard-coded where needed. This is the "skip the framework" option from the idea above.

- **Hand-written error translation.** `tgmdp_vendor_frontend/src/lib/client.ts:152-197` runs every failed request through `translateAxiosMessage`: `"Network Error"` becomes `網路錯誤，請檢查您的連線`, any message containing "timeout" becomes `請求逾時，請稍後再試`, and `Request failed with status code 401/403/404/500` become fixed Chinese strings. The admin console has its own copy at `tgmdp_web_frontend/src/shared/api/client.ts:180`. Two copies of one table, not a shared catalog.
- **Library-level localisation.** `tgmdp_web_frontend/src/plugins/zodConfig.ts:3` is `z.config(z.locales.zhTW())`: Zod's built-in validation messages switch to Traditional Chinese globally. This is the one place a real locale object is used.
- **Number formatting by locale.** `tgmdp_vendor_frontend/src/features/TGMDPVENb004/components/BasicInfo.tsx:28`: `new Intl.NumberFormat("zh-TW").format(numPrice)`. Most other formatting is done by hand.
- **Strings inside logic.** Business messages are literals in Java and TS: `MplusSendMes.java:80-83` builds an alert mail body `"告警訊息：M+推播異常，錯誤訊息如下：..."`; Spring controllers and DTOs return Chinese `message` text in `ApiResponse`. Changing language would mean editing code, not a catalog.
- **Server vs client text agree by accident.** The server returns its own Chinese message in the body, but the web client's axios interceptor replaces `error.message` with the generic translation (`client.ts:153-154`), so the specific server text may be lost to the user. That is a known pain: see [[Error Handling]].
- **Encoding.** The backend declares `sys.encode=UTF-8` and `upload.encode=UTF-8` (`application.properties:73,86`). The checked-in DB2 schema dumps (`TGMDP_all_schema.sql`) display garbled comments when read as UTF-8, a reminder that the legacy DB side is not UTF-8-clean (the comment text appears to be in a legacy non-UTF-8 encoding).
- **Legacy.** JSP pages carried Chinese text inline in markup. The rebuild preserves labels verbatim to match the legacy screens during parity checks, which is why Chinese text appears in tests and review documents.

## Why it matters

These apps serve Traditional Chinese users, so wording, encoding and translated errors are part of correctness. Adding a second language later would be a project, not a setting.

## Check yourself

If an English-speaking vendor were onboarded, name three kinds of places in this codebase where strings would have to change.

## Related

- [[Error Handling]] · [[Validation]] · [[Serialization and DTOs]] · [[Component-Based UI]] · [[Configuration and Environments]]
- [[Vendor Portal API Layer]] · [[Web Console API Layer]] · [[Vendor Portal State and Forms]] · [[Concepts MOC]]
