---
tags: [codebase, tgmdp-vendor-frontend]
---
# Vendor Portal State and Forms

Up: [[tgmdp_vendor_frontend]] · [[System Overview]]

The portal uses Zustand for state and react-hook-form for input, the same pair as the admin console. The interesting differences are in *how much* is global, how stores are cleaned up, and how forms report errors.

## Where state lives

| State | Store | Persisted? |
|---|---|---|
| Logged-in vendor: `username`, `userData` (profile, menu, roles, tasks, buttons) | `src/features/auth/store/index.ts` | yes, `auth-storage` (whole state) |
| Modal, page title, sidebar open, dark mode | `src/stores/useUIStore.ts` | yes, `app-ui` (whole state) |
| One screen's results, loading, error | `src/features/TGMDPVENxNNN/store/…` | no |
| What the user is typing | react-hook-form inside components | no |

See [[Client State Management]] and [[Store Pattern]].

## The auth store is deliberately thin

```ts
// src/features/auth/store/index.ts:15-34
export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      username: null,
      userData: null,
      setUsername: (username: string) => set({ username }),
      setUserData: (userData: UserData) => set({ userData }),
      logout: async () => {
        try { await authApi.logout(); }
        catch { /* Even if the API call fails, still clear local state */ }
        finally { set({ username: null, userData: null }); }
      },
    }),
    { name: "auth-storage" },
  ),
);
```

Compare with the console's 230-line `useAuthStore`. Here the store only remembers facts. The **login flow lives in components** (`LoginComponent`, `OtpComponent`) that call `authApi` directly and then `setUsername` / `setUserData`. The session check lives in `ProtectedRoute`. That is less centralised, but each piece is easy to follow. See [[Vendor Portal Routing Auth and Permissions]].

`logout` clears local state in `finally`, so the user is logged out locally even if the network call fails. That is the right default for a security action: fail towards "logged out". See [[Defensive Programming]].

### Button permissions as a hook

```ts
// src/features/auth/store/index.ts:48-51
export const useHasButton = (): ((buttonKey: string) => boolean) => {
  const buttons = useAuthStore((state) => state.userData?.buttons);
  return (buttonKey: string) => buttons?.includes(buttonKey) ?? false;
};
```

The keys look like `"TGMDPVENc001_browse"`, exactly legacy's `task_id + "_" + button`. The hook **fails closed**: no list means no buttons, matching legacy `buttonList.contains()`. It is a [[Custom Hooks|custom hook]] that hides the store shape from components.

## Feature stores and cleanup

`src/features/TGMDPVENc003/store/useTGMDPVENc003Store.ts` is typical:

```ts
// lines 23-39 (trimmed)
search: async (sn) => {
  if (get().isLoading) return;
  // Drop the previous result up front: a failed or in-flight query must never
  // leave the earlier query's tables on screen (#236 …).
  set({ isLoading: true, error: null, results: null });
  try {
    const data = await TGMDPVENc003Api.search(sn);
    set({ results: data, isLoading: false });
  } catch (err) {
    set({ error: err instanceof Error ? err.message : "搜尋失敗，請稍後再試", isLoading: false, results: null });
  }
},
```

Three habits worth copying:

1. **Re-entrancy guard.** `if (get().isLoading) return;` drops a double-click instead of firing two requests that could finish out of order.
2. **Clear stale results first.** The comment cites a real ticket (#236). Showing the previous query's rows while a new query runs, or after it fails, misleads the user.
3. **A `reset()` action, called on unmount.** `ListPage.tsx:39` does `useEffect(() => reset, [reset]);`. Returning `reset` from the effect makes React call it when the page unmounts. Module-level stores outlive pages, so without this the last result would flash on the next visit.

Most vendor feature stores expose `reset` (`TGMDPVENb001` to `b005`, `c003`, …). The console solves the same problem centrally with `registerResettableStore` at logout; the portal solves it locally at unmount, and globally with a full-page redirect on 401 ([[Vendor Portal API Layer]]). Both are valid; local cleanup catches more cases, central cleanup is harder to forget. Compare in [[Web Frontends Compared]].

## The global UI store as a message bus

`useUIStore` (`src/stores/useUIStore.ts:45-113`) holds one modal at a time. Any code can call `showModal(message, title, button, onClose)` or `showConfirmModal(message, onConfirm, …)`, and the single `<MessageModal />` mounted in the layout renders it. This replaces legacy `alert()` and `confirm()` calls with a styled dialog, and it is an [[Observer and Pub-Sub]] channel: the caller does not hold a reference to the modal component.

`OtpComponent.tsx:50` calls `useUIStore.getState().showModal(...)` inside a `catch` block, with the comment "bypass any potential stale closure". Inside an async callback, a value captured from a hook at render time may be outdated; `getState()` always reads the current store.

Observation: both persisted stores use `persist` **without `partialize`**, so everything serialisable is written to `localStorage`, including `modal` and `pageTitle` in `app-ui`. Functions are dropped by JSON, so a persisted modal would come back without its callbacks. Whether this causes a visible bug depends on timing; it is worth a `partialize` like the console's. See [[Caching]].

## Forms

Two styles coexist.

**1. react-hook-form + Zod**, used by every search form (about 12 files import `zodResolver`). The schema sits next to the form:

```tsx
// src/features/TGMDPVENc003/components/SearchForm.tsx:12-38 (trimmed)
const searchSchema = z.object({
  sn: z.string().min(1, "序號為必填").max(30, "序號不可超過30個字元"),
});
const { register, handleSubmit } = useForm<FormValues>({ resolver: zodResolver(searchSchema) });
const handleValidationError = (errors: FieldErrors<FormValues>) => {
  if (errors.sn) showModal(errors.sn.message ?? "序號為必填", "系統通知", "確 認");
};
```

Messages are written per field in Chinese. There is no global Zod locale (the console sets `z.locales.zhTW()`), so any rule without a custom message would show Zod's English default. Larger schemas live in a `validations/` folder, for example the registration form in `src/features/register/validations/applyModal.ts` (account suffix 4-10 alphanumerics, 10-digit mobile number, required vendor). See [[Validation]].

**2. react-hook-form without a resolver**, with hand-written checks. The login components use `useForm` only to read values, then check them imperatively:

```tsx
// src/features/auth/components/LoginComponent.tsx:57-68
const userId = getValues("username") ?? "";
const captcha = getValues("captcha") ?? "";
if (!userId.trim()) { showModal("請輸入帳號", "系統通知"); return; }
if (!captcha.trim()) { showModal("請輸入驗證碼", "系統通知"); return; }
```

This mirrors the legacy JavaScript one-to-one, which made parity easy to verify, but it duplicates logic a schema would express declaratively.

In both styles errors are shown in a modal, as legacy did, not inline under the field.

## Questions to check understanding

1. What would happen if `ListPage` did not return `reset` from its effect, and a vendor searched, left, and came back?
2. Why does `useHasButton` return a function instead of a boolean?
3. Rewrite the login checks as a Zod schema. What do you gain, and what parity risk do you take on?
4. Which state in `app-ui` should really be persisted?

## Related

- Up: [[tgmdp_vendor_frontend]] · [[System Overview]]
- Siblings: [[Vendor Portal API Layer]] · [[Vendor Portal Routing Auth and Permissions]] · [[Vendor Portal Feature Trace - TGMDPVENc003 Serial Number Query]]
- Compare: [[Web Console State and Forms]] · [[Web Frontends Compared]]
- Concepts: [[Client State Management]] · [[Validation]] · [[Authentication and Sessions]] · [[Caching]] · [[Internationalization]] · [[Component-Based UI]]
- Patterns: [[Store Pattern]] · [[Observer and Pub-Sub]] · [[Custom Hooks]] · [[Container-Presentational]]
- Practices: [[Defensive Programming]] · [[Separation of Concerns]] · [[SOLID]]
