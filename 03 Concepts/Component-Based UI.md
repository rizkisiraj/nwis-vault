---
tags: [concept]
---
# Component-Based UI

Up: [[Concepts MOC]]

## The idea

A component is a small, reusable piece of interface that owns its markup, behaviour and sometimes style. A screen is built by composing components, the way a sentence is built from words. React, Vue and native toolkits all follow this idea.

Components take inputs (props) and render output. Good ones are small, do one job and can be tested alone. Poor ones grow into giant files that mix fetching, state and layout.

Key points:

- **Props down, events up**: data flows from parent to child; children report changes through callbacks.
- **Composition over inheritance**: combine small pieces rather than extending big ones.
- **Reuse**: a table or dialog written once serves many screens.
- **Separation**: logic moves to hooks or stores, display stays in the component.

## In this codebase

- **Vendor portal, feature folder**: `tgmdp_vendor_frontend/src/features/TGMDPVENc003/` has `pages/ListPage.tsx`, `components/{SearchForm,CurrentStatusTable,TransactionLogTable}.tsx`, `store/useTGMDPVENc003Store.ts`, `api/`. `ListPage` pulls state and actions from the store, wires effects (permission check, reset on leave), and passes plain props down. `SearchForm` takes `onSearch`, `isLoading`, `browsePermitted` (props down, events up) and owns only its form via react-hook-form + a zod schema. See [[Vendor Portal Folder and Feature Structure]].
- **Reuse of shared pieces**: the same feature imports shared `components/Button`, `Table`, `Legend`, `PageTitle` rather than redrawing them.
- **Admin console** uses the same `features/<name>/{api,pages,store,schema}` shape (e.g. `tgmdp_web_frontend/src/features/user-management/account-management/`) with Ant Design components. See [[Web Console Folder and Feature Structure]].
- **AMS**: `src/pages/C001M1/c001m1Page.tsx:3,17` calls `useC001M1()` and only renders; the hook holds state and validation. See [[AMS State Hooks and Testing]].
- **React Native shell** uses the same model with `app/` routes plus `features/`, `components/`, `hooks/` ([[RN Shell App Structure]]).

## Legacy vs modern

Legacy JSP pages mixed markup and Java in one file (`legacy/tgmdp_vendor_legacy/TGMDP_VENDOR/TGMDPVENc003/TGMDPVENc003Q1.jsp:10-11` reads `session` beans inside the page itself), and the Cordova app's screens were HTML fragments with inline `onClick="a001M1.multi_Click('4')"`. Reuse there meant copy and paste.

## Why it matters

Component size and responsibility decide how easy a screen is to change and test. The `ListPage` store-plus-presentational split is also why parity fixes (see the long legacy comment in `ListPage.tsx`) stay local.

## Check yourself

In `ListPage`, which parts could you reuse in another module unchanged, and which are specific to `TGMDPVENc003`?

## Related

- [[Container-Presentational]] · [[Custom Hooks]] · [[Feature-Sliced Structure]] · [[Separation of Concerns]] · [[SOLID]] · [[Single Page Applications]]
- [[Concepts MOC]]
