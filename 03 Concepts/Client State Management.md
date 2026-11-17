---
tags: [concept]
---
# Client State Management

Up: [[Concepts MOC]]

## The idea

State is the data a UI needs to remember: who is logged in, what a form contains, which dialog is open. Client state management is the choice of where that data lives and how components read and change it.

Local state sits inside one component. Shared state lives in a store that many components subscribe to. The hard part is deciding what is truly shared, what is server data that should be refetched, and what must survive a reload.

Key points:

- **Local vs global**: keep state as close to its user as possible.
- **Store**: one object plus actions, such as a Zustand store, avoids passing props through many layers.
- **Persistence**: writing state to localStorage keeps it after reload but can leak or go stale.
- **Source of truth**: a store can be only a projection of something stored elsewhere.

## In this codebase

All web apps and both RN shells use Zustand 5.

- **Per-feature store.** `tgmdp_vendor_frontend/src/features/TGMDPVENc003/store/useTGMDPVENc003Store.ts` holds `results`, `isLoading`, `error` and actions `search`, `reset`. It guards double submits (`if (get().isLoading) return`) and clears old results before each search so a failed query never shows stale tables. `ListPage` calls `reset` on unmount because the store is module-global.
- **Persisted auth.** Vendor: `src/features/auth/store/index.ts` persists the whole `{username, userData}` under `auth-storage` in localStorage, and `userData.buttons` drives `useHasButton`. `ProtectedRoute` re-verifies the session with the backend via cookie. Admin console: `tgmdp_web_frontend/src/features/auth/store/useAuthStore.ts:221-229` persists a `partialize`d subset (`isAuthenticated`, `user`, `tokenExpiresAt`, `sidebarMenu`, `taskSys`, `permissions`), a deliberate choice of what survives reload. The cookie is the real session; the persisted copy is only a hint, and it can go stale. See [[Web Console State and Forms]], [[Vendor Portal State and Forms]].
- **UI store.** Both consoles persist a `useUIStore` (theme, sidebar). `tgmdp_web_frontend/src/app/store/useUIStore.ts:144` names it `app-ui` and clears toasts on rehydrate so they do not persist.
- **Shell: memory plus secure storage.** `tgmdp_vendor_rn/store/useAppLifecycleStore.ts` keeps `backgroundedAt` in memory but mirrors it to `expo-secure-store` so it survives process death. See [[tgmdp_vendor_rn]].
- **AMS**: small stores under `tgmdp-app-ams/src/stores/` (`userStore`, `taskIdStore`, ...), no `persist` middleware. See [[AMS State Hooks and Testing]].

## Legacy vs modern

The Cordova app kept everything in one global object, `_APP_DATA` (`ams2-app-legacy/AMS2/platforms/android/app/src/main/assets/www/js/GlobeValue.js:2`), with flags such as `MENU_FLAG`, `TIMEOUT_SEC`, `BACKGROUND_TIME`. JSP apps kept state in the server `session`. Zustand stores are the narrow, typed replacement.

## Why it matters

Wrong placement of state causes stale screens, lost input and security leaks. Persisting permissions client-side is fine for display only if the server still enforces them.

## Check yourself

The vendor auth store persists all of `userData`. What would the admin console's `partialize` approach change?

## Related

- [[Store Pattern]] · [[Observer and Pub-Sub]] · [[Caching]] · [[Authentication and Sessions]] · [[Custom Hooks]] · [[Component-Based UI]]
- [[Concepts MOC]]
