---
tags: [pattern]
---
# Feature-Sliced Structure

Up: [[Design Patterns MOC]]

## The idea
Group code by user-facing feature rather than by technical type, so everything for one screen (api, pages, store, types) sits in one folder. The opposite layout is "by layer" (`all-components/`, `all-stores/`, `all-api/`), where one change scatters across the tree.

```
src/features/TGMDPVENc003/
   api/  pages/  store/  components/  types/
```

## Why it helps
- A change to one feature touches one folder.
- Features can be added or removed cleanly.
- Truly shared things (UI kit, HTTP client) live outside, in `shared/`, `components/`, `lib/`.

## In this codebase
- **Vendor portal, one folder per legacy task code.** `tgmdp_vendor_frontend/src/features/TGMDPVENc003/` contains `api/ components/ pages/ store/ types/`, each with its own `__tests__`. The folder name is the legacy module code, so a reviewer can map new code to the old JSP module at a glance (this is what the parity reports rely on). Cross-feature code sits in `src/shared/`, `src/stores/`, `src/lib/client.ts`.
- **Admin console, grouped by business area, then feature.** `tgmdp_web_frontend/src/features/` has `auth/`, `record-query/`, `user-management/`, `system-management/` and more. `features/auth/` holds `api components pages schema store types utils`. `record-query/` nests sub-features (`app-login-logout-record-query`, `web-change-record-query`, ...) and has its own `routes.tsx`, so each area registers its own routes. Deep nesting such as `system-management/project-acceptance-management/audit-supervisor-sets-tasks/store/` exists because the menu itself is nested.
- **RN shell: slices for flows, plus layer folders.** `tgmdp_rn_app/features/` has `login/`, `menu/`, `webview/`, `memory-overlay/`; `features/menu/` splits into `hooks/ screen/ config/ utils/ types/`. But `hooks/`, `stores/`, `core/` stay at the top level because they are app-wide, so it is a hybrid.
- **Not sliced: AMS web layer.** `tgmdp-app-ams/src/` is layered by type (`hooks/ pages/ stores/ services/ components/`), with sub-folders per legacy module (`hooks/b001`, `pages/d001`, ...). Slicing happens only inside each layer.

Legacy vs modern: the legacy JSP app was organised by WebWork action (`TGMDPs012.java` + `TGMDP/TGMDPs012/*.jsp`), which is already feature-shaped; the rebuild preserves that grouping by naming folders after the codes.

## Check yourself
Why does `useUIStore` live in `src/stores/` rather than inside a feature folder? (Hint: who reads it?)

## Related
- [[Store Pattern]] · [[Container-Presentational]] · [[Custom Hooks]] · [[Separation of Concerns]] · [[Vendor Portal Folder and Feature Structure]] · [[Web Console Folder and Feature Structure]] · [[RN Shell App Structure]] · [[Design Patterns MOC]]
