---
tags: [pattern]
---
# Strategy

Up: [[Design Patterns MOC]]

## The idea
Define a family of interchangeable algorithms behind one interface and pick one at runtime (or configuration time). The framework says *what* must happen; you plug in *how*.

```
interface PasswordEncoder { encode(raw); matches(raw, enc) }
new Md5PasswordEncoder()   // chosen at config time
```

## Why it helps
- Open/Closed: add behaviour without editing framework code.
- Each variant is tested alone.
- Replaces long if/else chains.

## In this codebase
Strategy here is mostly *plugging into framework interfaces*, not hand-built families.

1. **Password hashing.** `security/Md5PasswordEncoder.java` implements Spring Security's `PasswordEncoder` (`encode` = MD5 hex digest, `matches` = case-insensitive compare). It is plugged in by `security/WebSecurityConfig.java:45-48`: `@Bean PasswordEncoder passwordEncoder() { return new Md5PasswordEncoder(); }`. Why MD5: existing user passwords were hashed that way in the legacy DB, so the new login must verify the same hashes. Swapping to bcrypt later means changing one bean (plus migrating data).
2. **WS-Security password callbacks.** `config/CxfConfig.java` builds `wssProperties(Object passwordCallback)` (`:301-308`, `props.put("passwordCallbackRef", passwordCallback)`) and passes a different callback per endpoint: `new BtsEsbPasswordCallback(...)` (`:113`), `nwisMdpPasswordCallback` (`:140`), `new SoapInboundPasswordCallback(...)` (`:155`). Same algorithm skeleton (verify a UsernameToken), different credential source per partner.
3. **Task decorator.** `ContextCopyingTaskDecorator implements TaskDecorator`, plugged into the executor (`config/AsyncConfig.java`). See [[Decorator]].

**A missed opportunity (verified).** `security/AuthTokenFilter.java` picks how to authenticate with URL checks: `requestUrl.contains("/tgmdpweb/")` (`:79`, again `:127`), plus negative checks for `/tgmdpvendor/`, `/registry/`, `/acptads/`, `/ceringads/` (`:103-118`). One `AuthStrategy` per client type would make each branch independently testable.

**Legacy contrast.** `legacy/TGMDP/src/com/sys/integration/DBDAO.java:146-160` chooses a paging algorithm per database with `if (type.equals(IniValue.ORACLE)) ... MSSQL ... DB2`, calling `searchORACLE/searchMSSQL/searchDB2` (`:170, :209, :259`). That is the if/else chain Strategy would replace; adding a database means editing `DBDAO`.

I did not find a project-owned `Strategy` interface with several implementations in `tgmdp_backend` or `tgmdp_jobs`; claims that `MODULE_URL_MAP` (`tgmdp_vendor_frontend/src/shared/constants.ts:20`) is a Strategy are wrong, it is a plain lookup table of URLs.

## Check yourself
Why can `WebSecurityConfig` change the hash algorithm without any controller changing?

## Related
- [[Template Method]] · [[Factory]] · [[SOLID]] · [[Interceptor and Filter Chain]] · [[WS-Security]] · [[tgmdp_backend Security and Auth]] · [[tgmdp_backend Design Patterns Spotted]] · [[Design Patterns MOC]]
