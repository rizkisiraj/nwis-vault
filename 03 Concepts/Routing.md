---
tags: [concept]
---
# Routing

Up: [[Concepts MOC]]

## The idea

Routing maps an address to the code that handles it. On a server, a URL such as `/orders/5` is matched to a controller. In a front end, the router matches the URL to a screen and decides what to show without asking the server.

A router also guards access: it can redirect anonymous users to login, or hide pages a role may not use. Client guards improve experience only; the backend must repeat the check.

Key points:

- **Route table**: a list of path patterns and their targets.
- **Parameters**: parts of the path, like an id, are passed to the handler.
- **Guards**: checks that run before a route renders.
- **Lazy loading**: a route can load its code on first visit.

## In this codebase

- **Vendor portal** (`tgmdp_vendor_frontend/src/router.tsx`): `createBrowserRouter`, one `path: "TGMDPVENc003"` style entry per module, each page behind `lazy: lazyLoad(() => import(...))` (line 426-434). Protected routes sit under `<ProtectedRoute>` (line 81-84). That guard (`src/components/ProtectedRoute.tsx`) redirects with `<Navigate>` when the store has no `userData`, and otherwise re-verifies with `authApi.getUserData()` using the httpOnly cookie. See [[Vendor Portal Routing Auth and Permissions]].
- **Admin console** (`tgmdp_web_frontend/src/app/router/routes.tsx`): a `<ProtectedRoute />` layout route (line 51) wraps `AppLayout` and children; its `ProtectedRoute.tsx` calls `hydrate()` on each pathname change and redirects to `/login` if unauthenticated. Per-feature `routes.tsx` files add `TaskRouteGuard` (e.g. `features/document-download-area/routes.tsx:13`), which allows a route only if the user has a button for that task and system, copying the legacy `FrontControllerFilter` rule. See [[Web Console Routing Auth and Permissions]].
- **AMS**: `tgmdp-app-ams/src/App.tsx:181-189` declares `<Route path="/c001m1" ...>` etc.; `main.tsx:28` wraps it in `HashRouter` because the bundle is loaded from a local file in the WebView.
- **Mobile shell**: `tgmdp_rn_app/app/[module].tsx` uses Expo Router; it redirects `/<module>` to `/webview/<module>?origin=menu`, or to `/menu` if the param is missing. Routing across modules is the shell's job; routing within a module belongs to the web app. See [[RN Shell App Structure]].

## Legacy vs modern

Legacy URLs were mapped on the server by WebWork: `legacy/tgmdp_vendor_legacy/src/WW_XML/TGMDPVENc.xml:53-62` declares namespace `/TGMDPVENc003` with actions `begin` and `browse` (the latter calls `TGMDPVENc003.browse()` then chains to `begin`, which renders a JSP). A servlet filter, `com/sys/filter/FrontControllerFilter.java:101` (`button.checkButton(action)`), did the guard. The Cordova app had no router: it swapped fragments with `$("#contentPage").load("./menu.html")` (`ams2-app-legacy/.../www/js/AMS2.js:293,566`). See [[WebWork Request Flow]] and [[WebWork Actions vs Spring Controllers]].

## Why it matters

Routing is where navigation, permissions and bundle size meet, so mistakes show up as broken links or exposed pages. The new guards mirror legacy rules but are only a UX layer; the real enforcement is in backend `@PreAuthorize`.

## Check yourself

What URL pattern does `begin` in `TGMDPVENc.xml` correspond to in the vendor SPA's route table?

## Related

- [[Front Controller]] · [[Single Page Applications]] · [[MVC]] · [[Authentication and Sessions]] · [[Interceptor and Filter Chain]]
- [[Concepts MOC]]
