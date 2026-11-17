---
tags: [codebase, tgmdp-web-frontend]
---
# Web Console State and Forms

Up: [[tgmdp_web_frontend]] · [[System Overview]]

The console keeps state in three places, and each has a clear job:

| Kind of state | Lives in | Example |
|---|---|---|
| Who is logged in, menu, permissions | one global Zustand store, persisted | `src/features/auth/store/useAuthStore.ts` |
| One screen's data, filter and loading flags | one Zustand store per feature | `useAppLoginLogoutRecordQueryStore.ts` |
| What the user is typing right now | react-hook-form, inside the page | `useForm({ resolver: zodResolver(schema) })` |
| UI preferences (dark mode, sidebar) | `src/app/store/useUIStore.ts`, persisted as `app-ui` | |

Keeping "typing" state in the form library and "server" state in a store avoids re-rendering the whole page on each keystroke, and keeps the store free of half-typed values. See [[Client State Management]].

## Zustand in one paragraph

Zustand is a tiny [[Store Pattern]] library. `create()` returns a React hook. The store is a plain object of state plus functions; functions call `set()` to change state, and every component that reads the changed slice re-renders. There is no Provider, no reducer and no action types. It is an [[Observer and Pub-Sub]] system under the hood: components subscribe, `set()` notifies.

## A feature store, line by line

`src/features/record-query/app-login-logout-record-query/store/useAppLoginLogoutRecordQueryStore.ts`:

```ts
// lines 24-58 (trimmed)
export const useAppLoginLogoutRecordQueryStore =
  create<AppLoginLogoutRecordQueryState>((set, get) => ({
    options: { organizations: [], departments: [], loginStatuses: [], apps: [], logSources: [] },
    filter: { page: 1, size: 20, startDate: undefined, endDate: undefined },
    data: null,
    loading: false,
    setFilter: (filter) => {
      set((s) => ({ filter: { ...s.filter, ...filter, page: 1 } }));
      get().fetchData();
    },
    setPage: (page) => {
      set((s) => ({ filter: { ...s.filter, page } }));
      get().fetchData();
    },
```

Things to notice:

- **The store owns the async flow.** `fetchData` (lines 60-72) sets `loading`, calls the service, stores `data` or `error`, and clears `loading` in `finally`. The page only calls `setFilter(values)`. This keeps the page a thin view, close to the [[Container-Presentational]] split, with the store acting as the container.
- **A new search resets to page 1** (line 49). Changing a filter while on page 7 would otherwise ask for a page that may not exist.
- **The store imports the service, not axios** (`import { appLoginLogoutRecordQueryService } from "../api"`). That one import is the seam tests mock.
- **Errors become strings in state** (`error: (error as Error).message || "Fetch Data Failed"`). The UI decides how to show them.

This shape (options, filter, data, loading, exporting, error + `initOptions`, `fetchData`, `setFilter`, `setPage`, `exportExcel`) repeats across most query screens. Learn it once and you can read dozens of features.

## The auth store: persisted, global, and careful

`src/features/auth/store/useAuthStore.ts` is the most important store. It uses the `persist` middleware so a page refresh does not log you out:

```ts
// useAuthStore.ts:220-230
{
  name: "auth-storage",
  partialize: (state) => ({
    isAuthenticated: state.isAuthenticated,
    user: state.user,
    tokenExpiresAt: state.tokenExpiresAt,
    sidebarMenu: state.sidebarMenu,
    taskSys: state.taskSys,
    permissions: state.permissions,
  }),
}
```

`partialize` chooses what goes into `localStorage`. Functions and the in-flight `hydratePromise` are left out on purpose: a Promise cannot be serialised.

Important: the token itself is **not** in this store. It is an HttpOnly cookie. What is persisted is only a *copy of facts about the session* (user profile, menu, permissions, expiry time). That is fine for UX, but it means the client's idea of "logged in" can be stale. `hydrate()` (lines 169-218) handles that:

1. If the persisted session is still within its token lifetime **and** the menu and permissions are loaded, skip the network call (lines 180-182).
2. If another `hydrate()` is already running, await the same promise instead of firing a second `/auth/me` (lines 184-187). This "single-flight" trick matters because `ProtectedRoute` calls `hydrate()` on every navigation.
3. Otherwise call `/auth/me`; on failure mark the user logged out.

`toTokenExpiresAt` (`src/features/auth/utils/token-helper.ts:6-9`) turns the server's relative `expiresIn` (minutes) into an absolute epoch-ms timestamp at the moment the response arrives, which sidesteps client/server clock skew.

### Clearing state on logout

An SPA does not reload on logout, so every in-memory store would still hold the previous user's search results. `purgeLogin()` (lines 151-167) clears auth state, purges cached ticket photos from IndexedDB, and calls `resetAllStores()`:

```ts
// src/shared/store/resettableStore.ts:15-19
export function registerResettableStore<T>(store: StoreApi<T>): void {
  const initialState = store.getInitialState();
  resetFns.add(() => store.setState(initialState, true));
}
```

This is a registry: each store opts in, and logout runs every registered reset. A `try/catch` around each reset (lines 23-29) means one broken store cannot block the rest, a small piece of [[Defensive Programming]].

**Caveat worth knowing:** at the time of writing only about five feature stores call `registerResettableStore` (for example `useBaseStationResetHistoryQueryStore`, `usePersonnelSettingOperationStore`). Most feature stores, including `useAppLoginLogoutRecordQueryStore`, do not, so their last result survives a logout until the page refetches. It is a good first refactoring exercise.

## Forms: react-hook-form + Zod

Pages build forms with react-hook-form and validate with a Zod schema through `zodResolver`:

```tsx
// AppLoginLogoutRecordQueryPage.tsx:41-52
const { register, handleSubmit, setValue, watch, clearErrors, formState: { errors } } = useForm({
  criteriaMode: "all", mode: "onBlur",
  resolver: zodResolver(appLoginLogoutRecordQueryScheme),
});
```

Why this pair?

- react-hook-form keeps inputs **uncontrolled** (it reads DOM values via `register`), so typing does not re-render the page.
- Zod gives a single schema that both **validates** and **transforms**, and `z.infer` derives the TypeScript type from it. One source of truth for the form type and the request type. See [[Validation]] and [[Type Systems]].

The schema for this screen (`schema/appLoginLogoutRecordQueryScheme.ts`) shows three Zod techniques worth learning:

1. `z.preprocess` normalises raw input before checking it. The `dateField` turns `""`, `null` and `Date` objects into either `null` or a `YYYY/MM/DD` string (lines 6-14).
2. `.superRefine` adds cross-field rules. Lines 53-77 build full date-times from `startDate`+`startTime` and `endDate`+`endTime` and reject an end that is not after the start, with the legacy message 日期必需早於….
3. `.transform` reshapes the validated value into what the API wants (lines 78-88).

Error messages are in Traditional Chinese because `src/plugins/zodConfig.ts` runs `z.config(z.locales.zhTW())` before anything else is imported (`src/main.tsx:2`). See [[Internationalization]].

Reusable validators live in `src/shared/utils/validator/` (for example `commaSeparatedArray`, `timeHHMM`), and schemas reused by many screens live in `src/shared/schema/`.

### Showing errors the legacy way

Legacy pages showed validation failures in a modal dialog, not inline. The console reproduces that in the `onError` callback passed to `handleSubmit` (`AppLoginLogoutRecordQueryPage.tsx:103-128`), which opens `useMessageModal().showModal(...)`. `useMessageModal` is itself a tiny Zustand store (`src/shared/components/MessageModal.tsx:21`), a neat example of using a store as an app-wide event bus for UI.

## Dependent selects

Many filters are "company → department" pairs. `useDependentSelect` (`src/shared/hooks/useDependentSelect.ts`) watches the parent field, filters the child options and clears the child value when the parent changes. It is a good example of a [[Custom Hooks|custom hook]] that packages form behaviour so pages stay declarative.

## Questions to check understanding

1. Why is `hydratePromise` excluded from `partialize`?
2. What would a user see if `setFilter` did not reset `page` to 1?
3. Where does the TypeScript type `appLoginLogoutRecordQueryFormData` come from, and what happens to it if you add a field to the schema?
4. Write down what `registerResettableStore` would need to be added to, so a logged-out user's APP-log results disappear.

## Related

- Up: [[tgmdp_web_frontend]] · [[System Overview]]
- Siblings: [[Web Console API Layer]] · [[Web Console Routing Auth and Permissions]] · [[Web Console Feature Trace - APP Login Logout Record Query]]
- Compare: [[Vendor Portal State and Forms]] · [[Web Frontends Compared]]
- Concepts: [[Client State Management]] · [[Validation]] · [[Type Systems]] · [[Authentication and Sessions]] · [[Caching]] · [[Internationalization]] · [[Component-Based UI]]
- Patterns: [[Store Pattern]] · [[Observer and Pub-Sub]] · [[Custom Hooks]] · [[Container-Presentational]] · [[Singleton]]
- Practices: [[Separation of Concerns]] · [[Defensive Programming]] · [[SOLID]]
