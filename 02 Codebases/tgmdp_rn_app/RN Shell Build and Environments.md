---
tags: [codebase, tgmdp-rn-app, practice]
---
# RN Shell Build and Environments

> How `tgmdp_rn_app` becomes an installable app: Expo "prebuild" generating the native projects, config plugins editing them, three backend environments crossed with two install identities, the debug-versus-release split, Metro, and the gates that run before anything ships.

Up: [[tgmdp_rn_app]] · [[System Overview]] · Code side: [[RN Shell App Structure]]

## Expo, but not Expo Go

There are two ways to use Expo. In the **managed** way you never see native code and run the app inside the Expo Go app. In the **prebuild** (also called continuous native generation) way, `expo prebuild` writes real `android/` and `ios/` projects from `app.config.js`, and you build them with Gradle and Xcode.

This repo uses prebuild, because it ships its own Kotlin and Swift modules (camera, scanner, SSO data service, root detection) that Expo Go cannot load. Every run and build script starts with `expo prebuild` (`package.json:8-32`). So:

- `android/` and `ios/` are **generated output**. Hand edits are lost on the next prebuild.
- Every native customisation must be written as a **config plugin** in `plugins/`, a JavaScript function that edits Gradle files, the manifest, the Podfile or `Info.plist` during prebuild.

| Plugin | What it changes |
|---|---|
| `withAndroidDebugBuildConfig.js` | Metro-or-bundled toggle for debug builds, ABI splits |
| `withPlatformProtection.js` | Restores hardened native files (root detection, `allowBackup`) |
| `withIOSQuerySchemes.js` | URL schemes iOS may query, so the shell can tell whether legacy apps are installed |
| `withCameraCallback.js`, `withNoAndroidSplash.js`, `withIOSModularHeaders.js` | Camera result wiring, JS-controlled splash, Pod settings |
| `withKtlint.js`, `withSwiftLint.js`, `withIOSTests.js` | Native lint and test targets |

The comments in `app.config.js` show why this matters. For example `allowBackup: false` must be set in the Expo config and not only in the manifest, because Expo's own plugin rewrites the manifest value after the custom plugin runs (`app.config.js:163-167`). Plugin order is behaviour.

## Two independent axes: backend and identity

```ts
// tgmdp_rn_app/app.config.js:44-58
const isDebugIdentity = process.env.APP_VARIANT === "debug";
const idSuffix = isDebugIdentity ? ".debug" : "";
// ...
const appName = `${isDebugIdentity ? "TGMDPtest" : "TGMDP"}${envLabel}`;
const iosBundleId = `com.tgmdp${idSuffix}`;
```

- **`APP_ENV`** = `development` | `staging` | `production` selects the **backend**. `app.config.js:6-25` reads `configs/app-config.<APP_ENV>.json` (API base URL, timeouts, module URLs, WebView pool policy) and exposes it as `Constants.expoConfig.extra`. `configs/env.ts` and `configs/feature-flags.ts` read from there.
- **`APP_VARIANT=debug`** selects the **install identity**: a `.debug` bundle id, the `TGMDPtest` name, a separate iOS app group and SSO URL scheme (`app.config.js:66-70`).

Keeping them separate means a tester can install a UAT build pointed at staging *next to* the production app. The app group matters most: legacy apps find the shared SSO session by app group id, so a debug build that shared the production group would read the real user's session (`README.md:140-146`). This is [[Environment Separation]] taken seriously, and it is why the label `(prod)` is forced onto a debug build that talks to production (`app.config.js:51-54`).

Secrets are kept out: iOS signing certificates and the Android release keystore are located through environment variables (`TGMDP_PROVISION_DIR`, `TGMDP_RELEASE_STORE_FILE`), never committed (`README.md:102-133`). See [[Secrets Management]].

## Debug versus release, and Metro

**Metro** is React Native's JavaScript bundler and dev server. In a debug build the app normally downloads its JS from Metro on your laptop, which gives fast refresh. In a release build the JS is compiled to Hermes bytecode and embedded in the APK/IPA.

The Android project decouples "debug build type" from "talks to Metro" with one Gradle property, injected by a plugin:

```js
// tgmdp_rn_app/plugins/withAndroidDebugBuildConfig.js:3-10
const METRO_DEBUG_FLAG_BLOCK = `def tgmdpUseMetroDebugProp = findProperty("tgmdpUseMetroDebug")
def tgmdpUseMetroDebug = tgmdpUseMetroDebugProp != null
    ? tgmdpUseMetroDebugProp.toString().toBoolean()
    : (System.getenv("npm_execpath") != null || System.getenv("npm_command") != null)
`;
const DEBUGGABLE_VARIANTS_LINE =
  '    debuggableVariants = tgmdpUseMetroDebug ? ["debug", "debugOptimized"] : []';
```

- Run through `npm`: Metro is on by default.
- Run Gradle directly: the debug APK embeds its JS and needs no Metro.
- `-PtgmdpUseMetroDebug=true|false` always wins.

Why bother? The UAT `.debug` APK (`npm run build:apk:staging:debug`) is built with `assembleDebug` so it can be signed with the local debug keystore, but it must still be a self-contained, non-debuggable app (`scripts/build-apk-production.sh:41-56`).

**Known trap.** If the flag ends up false on a debug build while `expo-dev-client` is installed, the app opens to a permanently blank screen: the dev launcher expects a dev-support manager the release-style build does not have, so no bundle loads at all. It looks like a Metro network problem but is not. Run with `ORG_GRADLE_PROJECT_tgmdpUseMetroDebug=true` (team note `rn-app-metro-debug-flag`).

`metro.config.js:27` sets `config.cacheVersion = process.env.APP_ENV`. Without it Metro would serve a cached bundle carrying the previous environment's `extra` values after you switch `APP_ENV`. Caches must be keyed on every input that changes output: see [[Caching]].

## Build outputs

| Command | Output |
|---|---|
| `npm run build:apk:<env>` | One APK per CPU ABI (`arm64-v8a`, `armeabi-v7a`, `x86`, `x86_64`), because the plugin enables ABI splits |
| `npm run build:apk:<env>:debug` | Same, `.debug` identity |
| `npm run build:ipa:<env>[:debug]` | Archive + signed `.ipa` in `ios/build/` (`scripts/build-ipa.sh` checks certificate and profile before the long archive step) |
| `npm run prebuild:<env>` | Regenerate native folders only |

CocoaPods is pinned through a `Gemfile` because Ruby 2.6's `json` gem rewrites ~100 `Podfile.lock` checksums differently (`README.md:148-172`). Reproducible builds need pinned tools, not only pinned libraries: [[Build Tools and Dependency Management]].

## OTA: two different things with the same name

1. **Expo Updates** could replace the shell's own JS remotely. `configs/ota-update-config.js` *refuses* to emit an enabled `updates` block unless the URL is HTTPS and manifest code signing is fully configured. Its header comment explains why: an update server can run code on every device, and TLS authenticates the connection, not the payload.
2. **Module bundles** (AMS, NWIS2) are downloaded by the shell's own loader with SHA-256 verification. That is the OTA path actually used day to day: [[WebView Pool and Bundle Loading]].

## Gates before code ships

Local, via Husky (`package.json:49`): lint (zero warnings) → credential scan → `tsc --noEmit` → `madge --circular` → Jest with 100% coverage.

CI (`.gitlab-ci.yml`) repeats the same gates on merge requests and on `develop`, and adds `npm audit` against an allowlist. The file's own header gives the reason: `git commit --no-verify` and the GitLab web IDE skip Husky, so a pre-commit hook alone is not an enforcement point. See [[Test Coverage Gates]] and [[Linting and Formatting]].

API contract tests (`npm run test:api-contract`, Vitest) call a real backend environment to catch drift between the shell's DTOs in `data/` and the server.

## Try it yourself

1. Without building, predict the app name and bundle id for `APP_ENV=staging APP_VARIANT=debug`. Check against `app.config.js:34-58`.
2. Find where `MODULE_URLS` from `app-config.<env>.json` becomes `FeatureFlags`. What does "migrated" mean for a module in this codebase?
3. Explain to a colleague why "it works with `npm run android`" and "it is blank with `./gradlew installDebug`" can both be true.

## Related

- [[tgmdp_rn_app]] · [[System Overview]] · [[Mobile Architecture Overview]]
- [[RN Shell App Structure]] · [[WebView Pool and Bundle Loading]] · [[Native Modules and Legacy Coexistence]]
- Concepts: [[Configuration and Environments]] · [[Build Tools and Dependency Management]] · [[Caching]] · [[Hybrid Mobile Apps]]
- Practices: [[Environment Separation]] · [[Secrets Management]] · [[Test Coverage Gates]] · [[Linting and Formatting]] · [[Feature Flags and Mock Toggles]] · [[Git Branching]]
- New: Continuous Native Generation · [[Over-the-Air Updates]]
