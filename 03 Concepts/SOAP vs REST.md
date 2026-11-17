---
tags: [concept]
---

# SOAP vs REST

Up: [[Concepts MOC]]

## The idea

SOAP and REST are two ways for programs to call each other over HTTP. SOAP wraps every call in an XML envelope with a formal contract (WSDL), strict schemas and standard extensions such as WS-Security. It is verbose but very explicit.

REST uses plain HTTP verbs on URLs and usually exchanges JSON. It is lighter, easier to debug with a browser or curl, and has no required contract, which makes it flexible but looser.

Key points:
- **Contract**: SOAP has a machine-readable WSDL; REST relies on documentation or OpenAPI.
- **Payload**: SOAP is XML only; REST is mostly JSON.
- **Errors**: SOAP uses faults; REST uses status codes. Enterprise systems often mix both and return 200 with a failure code in the body.
- **Tooling**: SOAP needs generated clients; REST needs just an HTTP library.

## In this codebase

Both live side by side in one backend, split by *who the caller is*.

- **SOAP = the ESB seam.** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/config/CxfConfig.java` publishes Apache CXF endpoints per WSDL (`endpoint.publish("/MDPService")`, line 167, and eleven more). WSDLs sit in `src/main/resources/wsdl/` (e.g. `9_MDPSoapHttpService.wsdl`, `1_AMSSoapHttpService.wsdl`). Generated clients call out, e.g. `port.syncTicketByDate(header, body)` (`tgmdp_jobs/.../SyncTicketAllData.java:79`). Binary payloads travel as MTOM attachments (`"mtom-enabled","true"`, `CxfConfig.java:291`), which is how ticket XML files arrive. Security is WS-Security: [[WS-Security]].
- **REST = our own clients.** Browser and app traffic is JSON under `/api/v1/...` with `@RestController`, bearer/cookie auth and an `ApiResponse<T>` envelope (`PersonalMaterialsImportController.java:24-27`). See [[HTTP and REST]]. Some ESB integrations are also REST/JSON (`SdOneAntApiService`, fake side `AntController`), with their own error code `20002`.
- **Errors compared.** SOAP side: failure is in the header (`ch.setStatus("F")`, `MdpSoapEndpoint`) while HTTP stays 200. REST side: `ApiResponse.getCode()` becomes the HTTP status. The fake ESB mirrors both styles: SOAP endpoints in `endpoint/`, REST handlers plus `B1ValidationHandler` in `controller/` and `validation/`.
- **Contract strictness shows.** Handlers match on namespace and element (`@PayloadRoot(namespace = NAMESPACE_URI, localPart = "syncTicketByDateRequest")`, `tgmdp_fake_esb/.../MdpEndpoint.java:73`), so a wrong namespace never reaches your code. The REST side only fails when Jackson cannot parse or Bean Validation rejects a field.
- **Safe XML parsing.** The SOAP file-callback turns off DTDs and external entities before unmarshalling (`MdpSoapEndpoint`, `XMLInputFactory.SUPPORT_DTD=false`), a classic XXE guard that REST/JSON does not need.
- **Legacy vs modern.** The legacy `ADS` backends were SOAP services too; the rebuild keeps SOAP for the ESB and adds REST for everything the team owns. See [[Legacy ADS Adapter Backends]].

## Why it matters

Knowing which style a partner uses tells you how to build the client, how to read errors and how to fake it in tests.

## Related

- [[HTTP and REST]] · [[Message Integration and ESB]] · [[Serialization and DTOs]] · [[Error Handling]] · [[Adapter]]
- [[What an ESB Is]] · [[Why a Fake ESB Exists]] · [[tgmdp_fake_esb]] · [[tgmdp_jobs]]
- [[Concepts MOC]]
