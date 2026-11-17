---
tags: [pattern]
---
# Custom Hooks

Up: [[Design Patterns MOC]]

## The idea
A React hook (`useXxx`) packages stateful logic (state, effects, subscriptions) so components stay about rendering. Hooks compose, and can be tested alone with `renderHook`.

```
function useVersionCheck() {
  const [blocked, setBlocked] = useState(false)
  useEffect(() => { /* check */ }, [])
  return blocked
}
```

## Why it helps
- Reuse logic across screens.
- Keeps components thin.
- Side effects get one named, testable home.

## In this codebase
- **RN shell: app-wide behaviour as hooks.** `tgmdp_rn_app/hooks/` holds `useAppInitialization`, `useAppStateSync`, `useSessionManager`, `useVersionCheck`, `useNetworkStatus`, `useLegacyBridge`. Its README says these hooks mostly render no UI and bind lifecycle, navigation and session. Big hooks are split by concern into files: `useAppStateSync.lifecycle.ts`, `.logout.ts`, `.refresh-scheduler.ts`, `.validation.ts`, and `useVersionCheck.{ts,helpers,error,version}.ts`. `useSessionManager.ts:47` reads the store through a selector: `useAuthStore((state) => state.logout)`.
- **Screen controller hooks.** `tgmdp_rn_app/features/menu/hooks/useMenuScreenController.ts` re-exports `useMenuScreenControllerInternal`, which composes `useMenuScreenDependencies`, `...LocalState`, `...Handlers`, `...Initialization`. It returns `{state, handlers}` for the view (see [[Container-Presentational]]).
- **WebView boot handshake.** `tgmdp_rn_app/core/webview/useWebViewBoot.orchestrator.ts:154` (`useWebViewBootOrchestrator`) wires lifecycle and effects hooks from `useWebViewBoot.effects.ts`. The AMS side mirrors it in `tgmdp-app-ams/src/hooks/useWebViewBoot.ts:29`, documented as "mount once, in App" because every mounted instance re-registers the shim handler and makes the host re-send BOOTSTRAP. That is a real hook pitfall the team wrote down.
- **Data-fetch hooks (AMS).** `tgmdp-app-ams/src/hooks/usePersonalIssues.ts` keeps `data/loading/error` in `useState`, fetches via `AMS_API.getTicketCountForUser`, and returns them. Folders `hooks/ticket`, `hooks/b001`, `hooks/gnb` group many `useGetXxx` hooks per legacy module.
- **Web console utilities.** `tgmdp_web_frontend/src/shared/hooks/` has small generic hooks: `useDebounce`, `useDebouncedCallback`, `useArrayPagination`, `useDependentSelect` (clears dependent form fields when a parent select changes).

Vendor portal: few custom hooks; logic goes into Zustand stores instead (`TGMDPVENc003/store/useTGMDPVENc003Store.ts`), see [[Store Pattern]]. Legacy jQuery apps had no equivalent; logic lived in page scripts and globals.

## Check yourself
Why would mounting `useWebViewBoot` in two components be a bug, even though hooks are "just functions"?

## Related
- [[Container-Presentational]] · [[Store Pattern]] · [[Observer and Pub-Sub]] · [[RN Shell App Structure]] · [[AMS Bridge Client]] · [[AMS State Hooks and Testing]] · [[Vendor Portal State and Forms]] · [[Design Patterns MOC]]
