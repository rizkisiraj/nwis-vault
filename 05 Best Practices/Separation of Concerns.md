---
tags: [practice]
---
# Separation of Concerns

Up: [[Best Practices MOC]]

## The idea
Split a program so each part handles one concern: presentation, business rules, data access, transport. A change to one concern should touch one place. Layers, feature folders and container/presentational components are all versions of this.

## Why it matters
- Bugs are easier to localise.
- You can test business rules without a browser or a database.
- Teams can change the UI or the DB without rewriting rules.

## In this codebase
- **Backend layers by package:** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/` has `controller/`, `service/`, `repository/`, `dto/`, `mapper/`, `security/`, `filter/`, `middleware/`, `exception/`. Both `controller/` and `service/` are then split by system (`nwisads`, `tgmdpweb`, `tgmdpvendor`, `amsads` ...). Cross-cutting work such as log masking lives in `middleware/LogFilter.java:38`, not in business code.
- **Web front end by feature:** `tgmdp_web_frontend/src/features/document-download-area/` has `api/`, `pages/`, `types/`, `routes.tsx`. HTTP is isolated in `src/shared/api/client.ts`, whose request interceptor (`:71-90`) alone trims empty params and builds the base URL.
- **Vendor portal:** `tgmdp_vendor_frontend/src/features/auth/` has `api/`, `components/`, `pages/`, `store/`, `types/`. Auth state is in `store/index.ts`, not in page components.
- **Transport vs UI in the RN shell:** `tgmdp_rn_app/configs/` splits axios concerns across `axios.request.ts`, `axios.response.ts`, `axios.auth-context.ts` and `axios.session-port.ts` instead of one big client.
- **Legacy vs modern:** legacy `legacy/TGMDP/src/com/sys/...` has `presentation`, `integration` and servlet code, but JSP and action classes still embed SQL and rendering decisions. The rebuild enforces the split by folder.

## Check yourself
Why does the web console keep the real/mock choice in `api/index.ts` rather than inside the page component?

## Related
- [[SOLID]] · [[Service Layer]] · [[Container-Presentational]] · [[Feature-Sliced Structure]] · [[tgmdp_backend Layering and Packages]] · [[Web Console Folder and Feature Structure]] · [[Best Practices MOC]]
