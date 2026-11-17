---
tags: [concept]
---

# Contract Testing

> Up: [[Concepts MOC]]

## The idea

A **contract** is the agreement about message shape between two systems: field names, namespaces, required fields, formats and status codes. Contract testing checks that both sides honour that agreement, without needing the whole real system running.

In the consumer-driven form (tools like Pact), the consumer records what it expects, and the provider is verified against those expectations. It sits between unit tests and full end-to-end tests: cheaper than the second, more realistic than the first.

A contract is only as good as the thing it is checked against. If the stand-in drifts from the real provider, the tests give false confidence.

## In this codebase

No Pact or similar tool is used. The contracts are the WSDL/XSD files and interface specs, and the checks are three home-grown things.

- **Wire tests against the production endpoint config.** `tgmdp_backend/src/test/java/.../service/amsads/AmsSoapWireTest.java` publishes the real `AmsSoapEndpoint` on a CXF `local://AMSService` address using the *same* settings as production (`CxfConfig.configureAmsEndpoint`), then sends a hand-built `wsse:Security` envelope (Timestamp + UsernameToken + PasswordDigest) with the right namespaces. Only the services behind the endpoint are Mockito mocks. So the SOAP contract (namespace, operation, security) is tested for real, the business code is not. `NwistgmdpSoapWireTest` does the same for ACPT, and `CxfConfig{Acpt,Ams,Nwis}EndpointTest` assert the interceptor action is `"UsernameToken Timestamp"`.
- **Fake ESB as a contract fixture.** `tgmdp_fake_esb` serves the real WSDLs (`config/SoapWebServiceConfig.java`, `SimpleWsdl11Definition`) and dispatches on `@PayloadRoot(namespace, localPart)` (e.g. `MdpEndpoint.java:73`), so a client with the wrong namespace or operation name is not routed. On the REST side Bean Validation on DTOs (`dto/ams_esb/B1EsbRequest.java`, `@NotBlank` on required fields) plus `B1ValidationHandler` (`validation/B1ValidationHandler.java:20-41`) return status `20002` for bad JSON or missing fields, reproducing the real ESB's response. See [[Why a Fake ESB Exists]] and [[Fake ESB Internals]].
- **Live-ESB tests exist but are tagged out.** `@Tag("esbIntegration")` (e.g. `IrtApiB1IntegrationTest`) runs only under the Gradle task `esbIntegrationTest` (`tgmdp_backend/build.gradle:142-146`); the default `test` task excludes it (line 99). That is the nearest thing to verifying the *real* provider.
- **The weak link.** Nothing automatically compares the fake's behaviour with the real ESB. If the real ESB changes, the fake and the tests keep passing. `ESB_MAPPING.md` (workspace root) is a manual table of that alignment, dated "verified 2026-06-17".
- **Frontend contract.** Types such as `ApiResponseDto<T>` (`tgmdp_vendor_frontend/src/types/ApiResponseDto.ts`) describe the expected JSON, but TypeScript types are erased at runtime, so nothing checks real responses; see [[Type Systems]].

## Check yourself

Which side of the contract does `AmsSoapWireTest` protect: the ESB-calls-backend direction or backend-calls-ESB? What would you add to cover the other?

## Related

- [[Mocking]] · [[Testing Pyramid]] · [[Unit Testing]] · [[Validation]]
- [[SOAP vs REST]] · [[tgmdp_fake_esb]] · [[Concepts MOC]]
