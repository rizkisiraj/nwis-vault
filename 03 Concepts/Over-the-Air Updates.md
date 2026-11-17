---
tags: [concept]
---

# Over-the-Air Updates

> Up: [[Concepts MOC]]

## The idea

An over-the-air (OTA) update changes part of an installed mobile app without going through an app-store release. The app downloads new code or assets from a server and uses them next time it starts.

It is fast, but it means a server can make every device run new code. TLS only proves *who you are talking to*, not that the payload is genuine, so a safe OTA scheme also verifies the content itself, through code signing or a hash check, and keeps a fallback if a download fails.

"OTA" can mean two different mechanisms in the same app, which the notes call out.

## In this codebase

- **Expo Updates (shell's own JS), guarded.** `tgmdp_rn_app/configs/ota-update-config.js` builds the Expo `updates` block and refuses to produce an enabled one unless the URL is `https://` (line 64) and code-signing certificate, key id and algorithm are all present (lines 66-86); otherwise it returns `{ enabled: false }`. The header comment explains why: a URL alone silently enables OTA in `@expo/config-plugins`, and TLS authenticates the connection, not the payload. See [[RN Shell Build and Environments]].
- **Module bundles (day to day).** `tgmdp_rn_app/core/bundled-manager/` downloads zips (`downloader.ts`), validates and extracts them, verifies SHA-256 (`verifier.ts`, uses `CryptoDigestAlgorithm.SHA256`) and installs transactionally (`install-transaction.ts`). The trusted checksum is the `X-SHA256` response header on the download (`loader.ts:33` comment), and a missing header fails closed. See [[WebView Pool and Bundle Loading]].
- **Seed bundle.** `tgmdp_rn_app/assets/modules/AMS/0.0.0.zip` ships in the app so AMS can open before any download; the vendor shell does the same in `core/seed-bootstrap/application/bootstrapSeedBundle.ts`.
- **Vendor shell flow.** `tgmdp_vendor_rn/core/webview/application/webViewSourceResolver.ts` bootstraps the seed, runs `loadModule` against a registry, and maps outcomes: `success_install` / `cache_hit` load the local path, `silent_skip` falls back to the cached copy, `fatal_dialog` shows a blocking restart dialog (`【檢查更新回傳：<code>，按下確認鍵後將自動重啟APP。】`). This is the "keep a fallback, fail closed on fatal" pattern.
- **Bridge operations.** `MODULE_UPDATE` and `CHECK_VERSION_REGISTRY` are in `tgmdp_rn_app/types/bridge/messageType.ts`; AMS sends `CHECK_VERSION_REGISTRY` (`tgmdp-app-ams/src/utils/bridge/api.ts:393`) and has a 60 s `MODULE_UPDATE_TIMEOUT_MS`. The AMS GitLab pipeline publishes the zip to a registry (`tgmdp-app-ams/.gitlab-ci.yml`, `npm run build-zip`).
- Trade-off discussion: [[Hybrid Shell Architecture Trade-offs]]. Bridge protocol: [[Bridge Engine and Message Protocol]].

## Legacy vs modern

Legacy Cordova used `cordova-hot-code-push-plugin`: the app's `www/chcp.json` points to a `content_url` and `chcp.manifest` lists every file with an MD5 hash, so the plugin could diff and download changed files (`ams2-app-legacy/AMS2/platforms/android/app/src/main/assets/www/chcp.manifest`). The modern scheme replaces per-file MD5 with whole-bundle SHA-256 and a version registry. Error codes still follow the old plugin's CHCP numbering: `tgmdp_vendor_rn/core/bundle-loader/domain/chcpClassifier.ts` classifies them (fatal vs silent), and notes it intentionally differs from `tgmdp_rn_app/core/bundled-manager` on some codes (-15, -19).

## Check yourself

Why does the config refuse `updates.enabled: true` with only a URL, when TLS already protects the download?

## Related

- [[Hybrid Mobile Apps]] · [[WebView and Native Bridge]] · [[Caching]]
- [[Concepts MOC]]
