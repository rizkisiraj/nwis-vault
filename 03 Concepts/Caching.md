---
tags: [concept]
---

# Caching

Up: [[Concepts MOC]]

## The idea

A cache stores the result of expensive work so it can be reused. It trades memory for speed and freshness: data may be out of date. The classic difficulty is deciding when a cached value is no longer valid.

Caches have a size limit, so an eviction rule decides what to drop. Least Recently Used (LRU) removes whatever was used longest ago.

Key points:
- **Where**: browser, app memory, a WebView, a CDN or a server.
- **Key**: what identifies a cached item, such as a request's parameters.
- **Invalidation**: expiry time, version number or explicit clear.
- **Risk**: stale or sensitive data persisting longer than intended.

## In this codebase

No Spring cache abstraction is used (grep found no `@Cacheable`, `@EnableCaching` or Caffeine). Caches are hand-written.

- **Backend parameter cache with a version check.** `tgmdp_backend/.../service/MediatorService.java` loads all parameter code-to-name pairs into two `HashMap`s at startup (`@PostConstruct`, lines ~44-47). `lookup()` first calls `validateCache()`, which compares `getLatestUpdateTimestamp()` from the DB with a stored `lastLoadedTimestamp` and reloads only if it changed (lines 50-72). Invalidation is "ask the DB if anything changed", one cheap query instead of a full reload per lookup. `lookupWithouValidate` skips the check for hot paths. A twin exists in `tgmdp_jobs/.../service/impl/MediatorService.java`.
- **Lazily loaded cache, double-checked locking.** `TgmdpvendorAccountService.sqeNames()` (~line 351-362) loads a SQL-error-name table once, under `synchronized`, mirroring a legacy `SQLErrorMediator.init()`. It never refreshes, so it is stale-by-design until restart.
- **Mobile LRU WebView pool.** `tgmdp_rn_app/modules/runtime/webview-pool/` keeps a small number of live WebViews in `strict-3-lru` mode (`policy.ts:263`). When the pool is full, `state.transition.ts:17-22` emits `evict` of the oldest *cached* item and `promote`s the new one; `state.shared.ts:77-84` (`selectOldestCachedIndex`) sorts cached items by age. Memory pressure trims further (`pushMemoryPressureOps`, line 7). This is eviction with a capacity of three, to avoid the OS killing the app.
- **Downloaded bundle cache.** The shell keeps verified web bundles on disk by version (`core/bundled-manager/`) so it only downloads when the registry has a newer one. See [[WebView Pool and Bundle Loading]] and [[Over-the-Air Updates]].
- **Browser persistence.** zustand `persist` writes the vendor auth store to localStorage under `auth-storage` (`tgmdp_vendor_frontend/src/features/auth/store/index.ts:16-32`). Useful, but it is a cache of identity that can outlive a server-side logout. See [[Vendor Portal State and Forms]].
- **Not a cache but looks like one.** `IniValue` / `IniValueService` hold config in static state loaded once (`ensureSysLoaded`, `synchronized`), with `setService.refresh()` called at the start of each job run (`JobExecuter.executeInternal`).

Legacy vs modern: the legacy app kept `UserBean` / `ButtonBean` in the HTTP session as a per-user cache of permissions; the rebuild keeps permissions in the token.

## Why it matters

A cache boosts speed on slow connections and devices, but stale data and leftover secrets are real bugs.

## Check yourself

`MediatorService` reloads when the *latest update timestamp* changes. What kind of edit to the parameter tables would it fail to notice?

## Related

- [[Client State Management]] · [[Hybrid Mobile Apps]] · [[File Upload and Storage]] · [[Configuration and Environments]] · [[Singleton]]
- [[Hybrid Shell Architecture Trade-offs]] · [[Concepts MOC]]
