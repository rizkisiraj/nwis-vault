---
tags: [codebase, tgmdp-rn-app]
---
# RN Shell App Structure

> How the code inside `tgmdp_rn_app` is organised: the routes that expo-router builds from the `app/` folder, the feature folders behind them, the Zustand stores, the root-level hooks, and the strict file-size and style rules that shape every file.

Up: [[tgmdp_rn_app]] · [[System Overview]] · Build side: [[RN Shell Build and Environments]]

## Routing: the file system is the router

expo-router turns files under `app/` into screens, the same idea as Next.js. The whole native navigation tree is small, because most screens live inside web modules:

```
app/
├── _layout.tsx            root layout: providers, splash, overlays, <Stack/>
├── index.tsx              redirect to /menu or /login
├── [module].tsx           /AMS2 → /webview/AMS2?origin=menu
├── login/index.tsx        credentials, OTP, device registration, biometrics
├── menu/index.tsx         module grid
├── webview/[moduleId].tsx hosts one module in a WebView
└── +not-found.tsx
```

The root index waits until the keychain restore has finished before it chooses a route. Without that wait a user with a valid session would see the login screen flash on cold start:

```tsx
// tgmdp_rn_app/app/index.tsx:8-17
const Index = () => {
    const navState = useRootNavigationState();
    const isAuthenticated = useAuthStore((state) => state.isAuthenticated);
    const isSessionRestored = useAuthStore((state) => state.isSessionRestored);
    const navKey = (navState as { key?: string } | null)?.key;
    if (typeof navKey !== 'string' || navKey.length === 0) return null;
    if (!isSessionRestored) return null;
    return <Redirect href={isAuthenticated ? '/menu' : '/login'} />;
};
```

`app/[module].tsx` exists so that a deep link or a menu tap always goes through one path, `/webview/<module>`. One entry point means one place to enforce the session gate, which is a small [[Front Controller]].

See [[Routing]].

## The root layout

`app/_layout.tsx:109-135` nests providers in a fixed order: `SafeAreaProvider` → `WebViewPoolProvider` → `ThemeProvider` → `<Stack/>` + `RootOverlayManager`. Two design choices stand out:

- **The WebView pool sits above the navigator.** WebViews survive route changes, so going back to the menu and opening AMS again reuses a live page instead of reloading it. See [[WebView Pool and Bundle Loading]].
- **Overlays are mounted once at the root.** The loading spinner, the employee-number watermark, the version dialog, network alerts and the dev FAB float above every screen, including WebViews. A comment at `app/_layout.tsx:116-118` warns that the `<Stack/>` must stay mounted even before the app is ready, or the layout re-mounts until React hits its depth limit.

The layout calls three root hooks (`app/_layout.tsx:32-53`): `useAppInitialization` (push, FCM token sync, notification taps), `useAuthNavigation` (react to sign-in/sign-out) and `useLegacyBridge` (deep links from legacy apps).

## Feature folders

`features/` groups code by user-facing feature rather than by technical type. That is [[Feature-Sliced Structure]]:

| Feature | Responsibility |
|---|---|
| `features/login` | Credentials, OTP, device registration, biometric login, recovery |
| `features/menu` | Module grid filtered by the user's `sysAuth`, logout, biometric binding, WebView pre-warm, deciding WebView vs legacy launch (`hooks/actions/menu-screen.launch-actions.ts:240-266`) |
| `features/webview` | The screen controller for one module: resolve the bundle source, run the boot handshake, wire the bridge, handle hardware back, block cross-origin navigation, route push notifications to the right module |
| `features/memory-overlay` | Dev-only memory monitor |

A feature is split into many small files with a dotted naming scheme, for example `useWebViewScreen.ts`, `useWebViewScreen.handlers.ts`, `useWebViewScreen.effects.ts`, `useWebViewScreen.security.ts`. Each file holds one concern of the same hook. This keeps every file inside the ESLint size and complexity limits and makes coverage easier to reach.

## State: Zustand stores

| Store | Holds |
|---|---|
| `stores/auth/useAuthStore.ts` | user, token, expiry, `sessionStatus`, `isLoggingOut`, pending session dialogs |
| `stores/auth/use-login-store/` | multi-step login flow state |
| `stores/notificationStore.ts` | pending push navigation for a module |
| `stores/useLoadingStore.ts` | global loader with **owners** so two tasks cannot hide each other's spinner |
| `stores/useGlobalModalArbiterStore.ts` | decides which global modal wins when several want the screen |
| `stores/useNetworkStore.ts`, `useHostPolicyStore.ts`, `useMenuHotfixStore.ts`, `useStartupNoticeStore.ts`, `useDevStore.ts`, `useWebViewPoolInspectorStore.ts` | smaller cross-cutting state |

The auth store's header comment (`stores/auth/useAuthStore.ts:1-8`) is an important lesson: it is an *in-memory projection* for the UI. The durable truth is secure storage plus the native USERFILE, managed by `services/auth/session/`. Keeping "what the UI shows" separate from "what is persisted" avoids a whole class of bugs where a re-render races a disk write. See [[Client State Management]] and [[Store Pattern]].

## Services: the façade layer

`services/` hides platform details behind small APIs:

- `services/auth/` login, biometrics, session manager, legacy launch coordination
- `services/bridge/` everything a bridge message can trigger (`APIRequest`, `APPCheckin`, `REQUEST_SESSION`, camera, scanner, reminder, OTA) plus resilience (retry, circuit breaker, rate limiter) and diagnostics
- `services/platform/` device info, location, intents, network, biometrics
- `services/notification/` FCM channels, foreground handling, tap navigation
- `services/storage/` secure storage with a migration step and a fresh-install guard

`BridgeNativeService` (`services/bridge/index.ts:53-94`) takes its collaborators through a constructor options object (`resolveDependencies(options.dependencies)`), which is how the tests swap in fakes: constructor [[Dependency Injection]].

## Root hooks

`hooks/` holds hooks that belong to the app, not to one feature: `useAppInitialization.*`, `useAppStateSync.*` (foreground/background, refresh scheduling, inactivity logout), `useVersionCheck.*` (forced update gate before anything else runs), `useNetworkStatus`, `useNotificationWebViewRouting`, `useLegacyBridge`. They are [[Custom Hooks]]: stateful logic with no JSX, reused by the root layout.

## House style enforced by tooling

- ESLint runs with `--max-warnings 0` (`package.json:34`), so a warning fails the build.
- `madge --circular` (`package.json:50`) fails on any import cycle. That is why `configs/axios.session-port.ts` exists (see [[Bridge API Request Flow and Auth]]).
- `lint:credentials` scans for hard-coded secrets (`scripts/scan-hardcoded-credentials.ts`).
- Tests live in `__tests__/`, mirror the source tree, and must keep coverage at 100%.

The repo also has a dedicated review agent (`rn-code-guardian`) whose brief lists these conventions: arrow-function components, no `any`, selectors for store reads, `StyleSheet.create` instead of inline styles.

## Questions to check understanding

1. Why is `WebViewPoolProvider` placed *outside* `<Stack/>` instead of inside the WebView screen?
2. `useLoadingStore` tracks an owner per spinner. Describe the bug that design prevents.
3. Pick one `useWebViewScreen.*.ts` file. What single concern does it own, and what would you lose by merging it back into `useWebViewScreen.ts`?

## Related

- [[tgmdp_rn_app]] · [[System Overview]] · [[Mobile Architecture Overview]]
- [[RN Shell Build and Environments]] · [[Bridge Engine and Message Protocol]] · [[WebView Pool and Bundle Loading]] · [[Native Modules and Legacy Coexistence]]
- Concepts: [[Routing]] · [[Client State Management]] · [[Component-Based UI]] · [[Dependency Injection]] · [[Layered Architecture]]
- Patterns: [[Feature-Sliced Structure]] · [[Store Pattern]] · [[Custom Hooks]] · [[Facade]] · [[Front Controller]] · [[Container-Presentational]]
- Practices: [[Clean Code Naming]] · [[Linting and Formatting]] · [[Separation of Concerns]] · [[Code Review]]
