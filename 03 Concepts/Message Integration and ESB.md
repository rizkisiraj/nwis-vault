---
tags: [concept]
---

# Message Integration and ESB

Up: [[Concepts MOC]]

## The idea

Real companies run many systems that were not built together. Integration is the work of making them exchange messages reliably. An Enterprise Service Bus (ESB) is a middleman in the centre: systems talk to the bus, and the bus routes, transforms and secures the messages.

The upside is that each system needs only one connection and one agreed message format instead of a connection to every other system. The downside is that the bus becomes critical infrastructure and a place where business rules can hide.

Key points:
- **Routing**: the bus decides which backend receives a message.
- **Transformation**: it converts formats, for example XML to JSON.
- **Security**: authentication, signing and WS-Security headers are handled at the edge.
- **Status vocabulary**: replies often return HTTP 200 even for business failures, so the body must be read.

## In this codebase

The ESB itself is a company system we do not own; the repos contain our two ends of it.

- **Outbound (we call the ESB).** Generated SOAP clients, e.g. `ESBPortType port = mdpEsbSoapService.getPort(); port.syncTicketByDate(header, body)` in `tgmdp_jobs/.../SyncTicketAllData.java:77-79`, with a `CommonHeader` (`messageFrom "MDP"`, `messageTo "ESB"`). Every outbound message gets WS-Security via `tgmdp_backend/.../service/esb/EsbWsSecurityHandler.java`. A few calls are REST/JSON (e.g. `service/sdoneads/SdOneAntApiService.java`, using `RestTemplateConfig`).
- **Inbound (the ESB calls us).** `CxfConfig.java:112-245` publishes about a dozen SOAP endpoints (`/AMSService`, `/MDPService`, `/FTSSoapHttpService`, `/ACPTSoapService`, `/IDMService`, ...), each with `WSS4JInInterceptor` and a `SysIdInInterceptor("AMS_ADS")`-style tag that records which logical subsystem the call belongs to. Contracts are the WSDLs in `tgmdp_backend/src/main/resources/wsdl/`.
- **Status vocabulary.** The reply header status is the truth, not HTTP: `MdpSoapEndpoint` sets `ch.setStatus("S")` or `"F"` (lines ~108-120); the fake's REST side uses `20002` for malformed input (`B1ValidationHandler.java:25`).
- **A stand-in for the bus.** `tgmdp_fake_esb` serves the real WSDLs (`config/SoapWebServiceConfig.java`, `SimpleWsdl11Definition` beans such as `"MDP"`, `"AMS"`), has one `endpoint/*Endpoint.java` per service (`@PayloadRoot` per operation, e.g. `MdpEndpoint.java:73`) and an `InboundTriggerController` so you can make the "ESB" call your backend. See [[Why a Fake ESB Exists]] and [[Fake ESB Internals]].
- **Mapping doc.** `ESB_MAPPING.md` at the workspace root lists each legacy adapter, the real upstream operation and the fake that stands in.
- **Legacy.** The `ADS` adapter backends (`legacy/ACPT_ADS`, `MRM_ADS`, ...) were the previous glue: [[Legacy ADS Adapter Backends]].

Anti-corruption in practice: the backend wraps each ESB service in its own package (`service/amsads`, `ftsads`, `btsads`, ...) so ESB vocabulary does not leak into controllers: [[Anti-Corruption Layer]].

## Why it matters

Most production bugs in this system sit at the seams with external systems, so understanding the bus explains many failures.

## Related

- [[SOAP vs REST]] · [[HTTP and REST]] · [[Serialization and DTOs]] · [[Adapter]] · [[Anti-Corruption Layer]] · [[Gateway]] · [[Mocking]]
- [[What an ESB Is]] · [[Integration Landscape]] · [[tgmdp_fake_esb]] · [[tgmdp_jobs]] · [[tgmdp_backend]] · [[tgmdp_backend ESB Integration]]
- [[Concepts MOC]]
