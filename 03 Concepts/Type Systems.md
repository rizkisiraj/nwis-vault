---
tags: [concept]
---

# Type Systems

Up: [[Concepts MOC]]

## The idea

A type system labels values as numbers, text, lists or custom shapes, and checks that code uses them consistently. Static checking, as in TypeScript or Java, catches mistakes before the program runs; dynamic checking waits until runtime.

Types also act as documentation and as a design tool. Strict settings forbid implicit `any`, unused variables and unsafe casts, so more mistakes are caught early.

Key points:

- **Static vs dynamic**: compile-time errors are cheaper than production ones.
- **Generics**: reusable types such as `ApiResponseDto<T>`.
- **Erased types**: TypeScript types vanish at runtime, so API data still needs validation.
- **Encoding rules**: a type can forbid a bad call, such as an unsupported HTTP method.

## In this codebase

- **Strict TypeScript everywhere in the modern frontends.** `"strict": true` plus `noUnusedLocals` and `noUnusedParameters` in `tgmdp_vendor_frontend/tsconfig.app.json:26-28`, `tgmdp_web_frontend/tsconfig.app.json:27-29` and `tgmdp-app-ams/tsconfig.app.json:64-66`; `tgmdp_rn_app/tsconfig.json:4` is also strict, and its ESLint turns `@typescript-eslint/no-explicit-any` to `'error'` (`eslint.config.js:179`).
- **Generics for the response envelope.** `tgmdp_vendor_frontend/src/types/ApiResponseDto.ts` defines `ApiResponseDto<T> = { success: true; data: T; RTN_DATA: T; ... }` and a separate error shape. Note the comment: both `data` and the legacy `RTN_DATA` key are typed, because the old and new backends disagree on the field name. The type records a migration. `tgmdp-app-ams/src/services/api/client.ts:30-41` wraps axios as `get<T>(...): Promise<ApiResponse<T>>`, so a caller gets typed data without casts.
- **A type that encodes a firewall rule.** `tgmdp_vendor_frontend/src/lib/client.ts:32-39` removes `put`, `patch` and `delete` from the axios instance type (`Omit<AxiosInstance, "get" | "post" | "put" | ...>`), so `apiClient.delete(...)` fails to compile. Reason in the comment: the WAF blocks those methods. Contrast the AMS client, which still exposes `put`/`delete`: a type is only as strict as its author chose.
- **Literal unions from a constant.** `tgmdp_rn_app/services/bridge/contracts/bridge-native.types.ts:30-39`: `SUPPORTED_BRIDGE_METHODS = [...] as const; type SupportedBridgeMethod = (typeof SUPPORTED_BRIDGE_METHODS)[number]`. One list is both the runtime whitelist and the compile-time type, so they cannot drift.
- **Erased types need runtime checks.** The web console pairs types with Zod 4 (`package.json`: `"zod": "4.6.7"`); about 90 files use `z.object`, and form types are derived with `z.infer` (e.g. `device-inquiry-operation/schema/deviceInquiryQuerySchema.ts:93`). `src/plugins/zodConfig.ts` sets the Traditional-Chinese locale. But API responses are mostly *trusted*: `ApiResponseDto<T>` is a promise, not a check.
- **Java side.** The backend is statically typed with DTO classes, `ApiResponse<T>` (`dto/ApiResponse.java:15`) and Bean Validation annotations (`@Valid`, `@NotBlank`) as the runtime check. DB2 is where types bite hardest: untyped bind parameters can be inferred wrongly by the database (SQLCODE -302 / -420), which Java's types cannot see. See [[DB2]].
- **Legacy.** The legacy front end is untyped jQuery/JSP, and the Java side passes `HashMap<String,Object>` rows (see `hm.get("BMP_ID").toString()` in `MplusSendMes.java:125-128`), so a wrong key fails at runtime. Even the modern jobs keep some of that style.

## Why it matters

Strict types let a developer change code with confidence, though they do not replace runtime validation of external data.

## Check yourself

`ApiResponseDto<T>` says `data` is a `T`. What happens at runtime if the backend sends `null` or a different shape, and which tool in this repo would catch it?

## Related

- [[Validation]] · [[Serialization and DTOs]] · [[Linting and Formatting]] · [[Defensive Programming]] · [[DTO Pattern]]
- [[Vendor Portal API Layer]] · [[Vendor Portal Build Lint and Testing]] · [[Bridge Engine and Message Protocol]] · [[tgmdp-app-ams]]
- [[Concepts MOC]]
