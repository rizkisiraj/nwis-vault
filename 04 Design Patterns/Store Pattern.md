---
tags: [pattern]
---
# Store Pattern

Up: [[Design Patterns MOC]]

## The idea
Keep shared client state in a central store with actions that change it; components subscribe to slices. State logic leaves the components. This codebase uses Zustand everywhere.

```
const useUIStore = create(set => ({ modal: null, showModal: m => set({modal:m}) }))
const modal = useUIStore(s => s.modal)   // selector
```

## Why it helps
- One source of truth.
- Selectors limit re-renders.
- Async actions (fetch, then set) live next to the state.

## In this codebase
- **Global UI store (vendor).** `tgmdp_vendor_frontend/src/stores/useUIStore.ts:46` is `create<UIState>()(persist(...))`; it holds page title, a single `modal`, dark mode and sidebar state with `showModal`/`closeModal` actions. `persist` with `name: "app-ui"` (line 104) saves preferences to localStorage.
- **Per-feature store with async action.** `tgmdp_vendor_frontend/src/features/TGMDPVENc003/store/useTGMDPVENc003Store.ts` holds `results, isLoading, error`; `search()` guards double-submit, clears the old result first ("a failed or in-flight query must never leave the earlier query's tables on screen"), awaits the API, then sets state. The page's `useEffect(() => reset, [reset])` clears it on leave.
- **Reset-on-logout registry (web console).** `tgmdp_web_frontend/src/shared/store/resettableStore.ts:15` `registerResettableStore(store)` captures `getInitialState()`; `resetAllStores()` (called from `useAuthStore.ts:166` in `purgeLogin`) restores every feature store. Why: the SPA stays mounted across logout/login, so without it one user's data would leak to the next. Persisted stores (`useAuthStore`, `useUIStore`) deliberately do not register.
- **RN shell and AMS.** `tgmdp_rn_app/stores/` has `auth/useAuthStore.ts`, `useLoadingStore.ts` (an `owners[]` list so each flow's `hide()` only releases its own `show()`), `useGlobalModalArbiterStore.ts`. `tgmdp-app-ams/src/stores/loadingStore.ts` uses a simple counter instead. Hooks read with selectors, e.g. `useAuthStore((state) => state.logout)`.

Legacy vs modern: legacy WebWork kept state in the server HttpSession; the SPAs keep it in browser memory, hence the logout-reset concern.

## Check yourself
Why is `useUIStore` allowed to survive logout but a feature store is not?

## Related
- [[Observer and Pub-Sub]] · [[Custom Hooks]] · [[Singleton]] · [[Client State Management]] · [[Vendor Portal State and Forms]] · [[Web Console State and Forms]] · [[RN Shell App Structure]] · [[Design Patterns MOC]]
