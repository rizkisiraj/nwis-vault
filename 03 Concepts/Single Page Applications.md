---
tags: [concept]
---
# Single Page Applications

Up: [[Concepts MOC]]

## The idea

A single page application (SPA) loads one HTML page and a bundle of JavaScript once. After that, JavaScript draws screens and fetches only data from the server (usually JSON), so moving between screens does not reload the page.

This feels fast and app-like, but the first load is heavier, the browser must run a lot of code, and jobs the server used to do, such as routing and login checks, move to the client. The server must still enforce permissions, because anyone can edit client code.

Key points:

- **Client-side routing**: the URL changes without a page reload.
- **Code splitting**: each page is loaded lazily to keep the first download small.
- **API-driven**: the server returns data, not HTML.
- **Client state**: the browser holds the data the UI needs between calls.

## In this codebase

All four web apps are Vite + React 19 + `react-router-dom` 7 + Zustand 5 SPAs.

- **Admin console** (`tgmdp_web_frontend/src/main.tsx`) mounts one React root; `app/App.tsx` just renders `AppRouter` (`src/app/router/routes.tsx`). Pages are mostly imported eagerly; lazy loading appears in a few feature route files (e.g. `features/system-basic-parameter-settings/app-module-update/routes.tsx:16`). See [[Web Console Routing Auth and Permissions]].
- **Vendor portal** (`tgmdp_vendor_frontend/src/router.tsx`) code-splits almost every page with React Router's `lazy:` plus dynamic `import()` (e.g. line 430-434 for `TGMDPVENc003`). Its `lazyLoad` wrapper (lines 15-31) reloads the page once if a chunk 404s, because a new deploy invalidates old hashed filenames; a `sessionStorage` timestamp stops infinite reload loops. This is a typical SPA-only failure mode. See [[Vendor Portal Routing Auth and Permissions]].
- **AMS** (`tgmdp-app-ams/src/main.tsx:3,28`) is an SPA that uses `HashRouter`, and `src/App.tsx:29` lazy-loads pages. It runs inside a WebView in the mobile shell, where the bundle is served from a local file path, so URLs with `#` avoid needing a server to resolve paths. See [[tgmdp-app-ams]].

## Legacy vs modern

The legacy monolith returned a full JSP page per request ([[Server-Side Rendering]]). The legacy Cordova app was a different shape again: one shell page, `AMS2.html`, that replaced a `#contentPage` div with HTML fragments (`ams2-app-legacy/AMS2/platforms/android/app/src/main/assets/www/js/AMS2.js:566`, `$("#contentPage").load("./a001/a001M1.html", ...)`). That is SPA-like navigation with no framework and global state.

## Why it matters

SPAs changed where logic lives and how login, errors and permissions must be handled. Compare [[Web Frontends Compared]] and [[Vendor Portal Build Lint and Testing]] for how the bundles are built.

## Check yourself

Why does the vendor portal need a chunk-reload guard, while a JSP site never would?

## Related

- [[Server-Side Rendering]] · [[Routing]] · [[Client State Management]] · [[Component-Based UI]] · [[Client-Server Architecture]] · [[Authentication and Sessions]]
- [[Concepts MOC]]
