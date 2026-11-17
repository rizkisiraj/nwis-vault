---
tags: [pattern]
---
# Container-Presentational

Up: [[Design Patterns MOC]]

## The idea
Split components into containers (fetch data, hold state, wire handlers) and presentational components (receive props, render). Presentational parts know nothing about stores or APIs. Modern React often moves the container's logic into a custom hook, leaving a thin route component.

```
<Page>  (container: store + api)
  <SearchForm onSearch isLoading />   (props only)
```

## Why it helps
- Presentational parts are reusable and trivially testable.
- Data flow is easy to follow.

## In this codebase
- **Vendor portal, classic form.** `tgmdp_vendor_frontend/src/features/TGMDPVENc003/pages/ListPage.tsx:11` pulls `results, isLoading, search, ...` from `useTGMDPVENc003Store()`, runs the effects (permission check, error redirect, reset on unmount), then renders `<SearchForm onSearch={...} isLoading browsePermitted />`, `<CurrentStatusTable data=... />`, `<TransactionLogTable ... />`. `components/SearchForm.tsx:18-22` takes only `onSearch`, `isLoading`, `browsePermitted` as props. It does own its form state (react-hook-form + zod), but it never touches the feature store. Purity here means "no API/store knowledge", not "no state".
- **RN menu, hook-as-container.** `tgmdp_rn_app/features/menu/screen/MenuScreenRoute.tsx:6-9` is the container: `const controller = useMenuScreenController(); return <MenuScreenView state={controller.state} handlers={controller.handlers} />`. `MenuScreenView.tsx:21-30` takes only `state` and `handlers` props and renders. So the container is a 5-line shell; all logic is in hooks ([[Custom Hooks]]).
- **Blurry edges.** `SearchForm` still imports `useUIStore` for modals (line 10), so a few "presentational" components reach into a global UI store. The pattern is followed as a guideline, not enforced.

Legacy vs modern: in JSP the server rendered HTML and the Action was the controller (see [[MVC]]); the container/presentational split is a client-side re-creation of that separation.

## Check yourself
In `ListPage`, which parts would you have to mock to unit-test `SearchForm`, and which to test `ListPage`?

## Related
- [[Custom Hooks]] · [[Store Pattern]] · [[Feature-Sliced Structure]] · [[MVC]] · [[Vendor Portal Folder and Feature Structure]] · [[Vendor Portal Feature Trace - TGMDPVENc003 Serial Number Query]] · [[RN Shell App Structure]] · [[Design Patterns MOC]]
