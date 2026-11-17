---
tags: [concept]
---

# WS-Security

> Up: [[Concepts MOC]]

## The idea

SOAP messages travel over plain HTTP, so the *message itself* needs a way to prove who sent it. WS-Security is a SOAP standard that adds a security block to the message header. The usual form is a **UsernameToken**: a username, a password (or a digest of it), a nonce and a timestamp.

Sending a digest instead of the password means the secret never crosses the wire. The nonce and timestamp stop an attacker from replaying an old message. The receiver recomputes the digest and rejects anything that does not match or is too old.

It lives in the message, not in the transport, so it works the same through intermediaries such as an ESB. That is also why it is a SOAP-era idea: modern REST APIs usually use bearer tokens in HTTP headers instead (see [[SOAP vs REST]]).

## In this codebase

WS-Security is real and used in **both directions**.

- **Inbound (ESB calls the backend): validate.** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/config/CxfConfig.java` publishes a dozen SOAP endpoints (`/AMSService`, `/MDPService`, `/FTSSoapHttpService`, ...) and puts a `WSS4JInInterceptor` on each (lines 112, 139, 154, 179, 192). All share one property set:

```java
props.put("action", "UsernameToken Timestamp");
props.put("passwordType", "PasswordDigest");
props.put("addUTElements", "Nonce Created");
props.put("passwordCallbackRef", passwordCallback);   // CxfConfig.java:301-309
```

  WSS4J does the digest/nonce/timestamp maths. The app only supplies the *known password* through a `CallbackHandler`, one per endpoint, e.g. `utils/SoapInboundPasswordCallback.java:19-30` (`if (account.equals(pc.getIdentifier())) pc.setPassword(password)`). Other callbacks: `BtsEsbPasswordCallback`, `NwisMdpPasswordCallback`, `DataCenterPasswordCallback`.
- **Outbound (backend or jobs call the ESB): generate.** `service/esb/EsbWsSecurityHandler.java` is a JAX-WS `SOAPHandler` that, for every outbound message, adds `Timestamp` (Created + Expires = now + 5 minutes), a random 16-byte nonce and a `PasswordDigest`. The old vault note missed this side.
- **Fake ESB: generate only.** `tgmdp_fake_esb/.../service/trigger/SoapTriggerService.java:234-239` builds the same token by hand: `Base64(SHA1(nonce + created + password))`, so a developer can fire a realistic ESB-to-backend call. A grep found no WSS4J validation in the fake, so it does **not** verify the tokens the backend sends it. See [[Fake ESB Internals]].
- **Tests prove the contract.** `AmsSoapWireTest` and `NwistgmdpSoapWireTest` build a `wsse:Security` header and push it through the real production endpoint config; `CxfConfig*EndpointTest` asserts the action string is `"UsernameToken Timestamp"`.
- **Why digest and SHA-1?** It is what the TWM ESB demands; the code comments say so. The setting `isBSPCompliant=false` shows the team relaxed the strict profile to interoperate.

Newer REST integrations do not use it; they rely on JWT/cookie auth (see [[Stateless vs Stateful]]).

## Check yourself

The backend validates a digest but never stores the ESB password in a user table. Where does the expected password come from at the moment WSS4J asks for it?

## Related

- [[SOAP vs REST]] · [[Message Integration and ESB]] · [[Authentication and Sessions]]
- [[Why a Fake ESB Exists]] · [[What an ESB Is]] · [[Concepts MOC]]
