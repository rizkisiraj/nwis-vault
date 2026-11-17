---
tags: [concept]
---
# Server-Side Rendering

Up: [[Concepts MOC]]

## The idea

With server-side rendering (SSR) the server builds the full HTML page for each request and sends it ready to display. The browser mostly shows it and sends a form back. Classic JSP, PHP and Rails apps work this way.

Modern frameworks (Next.js and others) can also render React on the server for fast first paint and search indexing, then take over in the browser.

Key points:

- **Pros**: simple mental model, fast first view, little client code.
- **Cons**: every navigation is a full round trip and the server holds UI logic.
- **State**: session state usually lives on the server.
- **Contrast**: an SPA sends data only and renders on the client.

## In this codebase

SSR in the classic sense exists only in the legacy system; none of the modern apps here render on the server (they are Vite SPAs with no server runtime).

- **Legacy page**: `legacy/tgmdp_vendor_legacy/TGMDP_VENDOR/TGMDPVENc003/TGMDPVENc003Q1.jsp` reads `UserID` and `ButtonBean` from the HTTP `session` (lines 10-11) and emits the tables as HTML inside scriptlets, only if the request attributes are set (`if(alView_STAT != null || alView_LOG != null)`, around line 44).
- **Action fills the view**: `legacy/tgmdp_vendor_legacy/src/com/prj/actions/tgmdpvenc/TGMDPVENc003.java:82-91` calls the backend, puts results into `request.setAttribute("view_stat", ...)` / `"view_log"`, and the framework forwards to the JSP. Routing: `src/WW_XML/TGMDPVENc.xml:53-62`. Server holds both logic and view. See [[WebWork Request Flow]].
- **Rebuilt as SPA**: the same screen is `tgmdp_vendor_frontend/src/features/TGMDPVENc003/pages/ListPage.tsx`, which fetches JSON and renders `CurrentStatusTable` and `TransactionLogTable` in the browser. Its comment records a parity detail inherited from the JSP: both result panels render after any search, even with zero rows.
- **Not SSR, but related**: the legacy Cordova app loaded static HTML fragments into a page (`$("#contentPage").load(...)`) and filled them with jQuery; the HTML was client-assembled, not server-rendered.

## Why it matters

Understanding SSR explains how legacy screens behaved (server session, request attributes, full reload) and why rebuilding them as SPAs changed validation, state and navigation. See [[Legacy TGMDP Monolith]], [[Legacy Systems]] and [[Server-rendered JSP vs React SPA]].

## Check yourself

Where did the legacy page get the logged-in user from, and where does the SPA get it?

## Related

- [[Single Page Applications]] · [[MVC]] · [[Front Controller]] · [[Strangler Fig]] · [[Authentication and Sessions]]
- [[Concepts MOC]]
