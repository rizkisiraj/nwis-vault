---
tags: [concept]
---

# Mocking

Up: [[Concepts MOC]]

## The idea

Mocking means replacing a real dependency with a stand-in during a test or development. A mock, stub or fake returns known answers so the code under test can run without a database, network or third-party system.

It makes tests fast and deterministic, but a fake that drifts from the real system gives false confidence. Mocks of whole services need to be checked against the real contract.

Key points:

- **Stub**: returns canned data.
- **Mock**: also records calls so a test can assert them.
- **Fake**: a working lightweight version, such as an in-memory store or a fake ESB.
- **Toggle**: a flag that switches an app between real and mocked data.

## In this codebase

Four different levels of stand-in are used:

- **Mockito mocks in unit tests (backend).** `AmsSoapWireTest` declares `@Mock AmsTicketSyncService`, `@Mock AmsTicketSyncRepository`, ... and `@InjectMocks AmsSoapEndpoint endpoint` (`tgmdp_backend/src/test/java/.../service/amsads/AmsSoapWireTest.java`), then `verify(...)` / `verifyNoInteractions(...)` to assert what the endpoint did. That is mocks in the strict sense: collaborators that record calls.
- **A fake whole service: the fake ESB.** `tgmdp_fake_esb` is a standalone Spring Boot app imitating the company ESB, with canned answers (`MdpEndpoint.syncTicketByDate` only logs and acks, `MdpEndpoint.java:73-78`; `queryTicketDetailList` returns a built-in mock ticket) and per-domain mock services (`service/slm/SlmMockService`, `service/smr/SmrMockService`, `service/wpu/WpuMockService`, ...). Developers point the backend at it instead of the unreachable real ESB. Why it exists and its limits: [[Why a Fake ESB Exists]], [[Fake ESB Internals]].
- **Environment toggle in the web console.** `tgmdp_web_frontend/src/features/mobile-device-management/message-push/api/index.ts:4-11`:

```ts
const USE_MOCK = import.meta.env.VITE_USE_MOCK === "true";
export const messagePushService = USE_MOCK ? messagePushApiMock : messagePushApi;
```

  Each feature has `xxxApi.ts` and `xxxApi.mock.ts` with the same shape and one switch at the export. The same pattern repeats across features (`document-download-area`, `device-inquiry-operation`, ...). See [[Feature Flags and Mock Toggles]].
- **Module mocks in frontend tests.** `tgmdp-app-ams/src/services/api/__test__/client.test.ts:3` uses `vi.mock('axios', ...)` to replace the HTTP library, so the client wrapper is tested without a network.
- **Test data via H2.** Jobs and some backend tests use an in-memory H2 DB in place of DB2 (`tgmdp_jobs/build.gradle:196`, `tgmdp_backend/build.gradle:303`), a fake rather than a mock. Real DB2 quirks (type-inference errors) can therefore hide from these tests.
- **Drift risk, concretely.** The fake ESB's answers (`20002`, `S`/`F` headers) were written by hand and nobody verifies them against the real ESB: [[Contract Testing]].

## Why it matters

Mocking is what makes unit tests possible and local development independent of external systems, at the cost of drift.

## Check yourself

In `AmsSoapWireTest`, which parts are real and which are mocked? What bug could still reach production?

## Related

- [[Unit Testing]] · [[Testing Pyramid]] · [[Dependency Injection]] · [[Feature Flags and Mock Toggles]] · [[Adapter]]
- [[Vendor Portal API Layer]] · [[tgmdp_fake_esb]] · [[Concepts MOC]]
