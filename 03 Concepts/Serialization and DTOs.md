---
tags: [concept]
---
# Serialization and DTOs

Serialization turns in-memory objects into bytes or text (JSON, XML, SOAP) to send or store; deserialization reverses it. A DTO (data transfer object) is a plain class shaped for the wire rather than for the database or UI, so internal models can change without breaking callers.

Typical bugs are mismatched field names, null versus missing, date formats and number precision.

## In this codebase
Three wire formats meet in the backend: JSON for the SPAs and apps, SOAP/XML for the ESB, and database rows.
- **A uniform JSON envelope.** `tgmdp_backend/src/main/java/com/taiwanmobile/boilerplate/dto/ApiResponse.java:10-30` is a generic `ApiResponse<T>` (`code`, `success`, `message`, `timestamp`, `data`, `error`, `meta`) annotated `@JsonInclude(NON_NULL)` so empty fields vanish from the JSON. Entities (`model/`, `sharedModules/.../model/`) are never returned to clients; DTOs live in `dto/<area>/`.
- **Legacy field names pinned with annotations.** App contracts keep upper-case names: `dto/nwisads/NwisPatrolStartResponseDto.java:19-30` maps `status` to `@JsonProperty("RTN_STATUS")` and `message` to `RTN_MESSAGE`, so Java stays camelCase while the wire matches the old app. In the other direction `@JsonAlias("TWM_SITE_ID")` (`dto/nwisads/tstSiteMeet/SiteMeetAddRequestDto.java:19`; ~37 files use `@JsonAlias`) accepts old and new names.
- **Row-to-DTO mapping.** Hand-written SQL often returns `Map<String,Object>` rows, which MapStruct converts: `mapper/tgmdpweb/resourceExchange/ResourceExchangeTransmissionStatsMapper.java` is `@Mapper(componentModel = "spring")` with `@Mapping(target = "dataArea", source = "DATA_AREA", ...)`. The uppercase DB column names stay inside the mapper.
- **XML for the ESB.** DTOs such as `dto/amsads/esb/AmsAccountAuthorityType.java:10-19` use `jakarta.xml.bind` annotations (`@XmlElement(required = true)`), and `jaxb/DataHandlerAdapter.java` is an `XmlAdapter` for `DataHandler` (attachment) fields. The same business data has a JSON DTO for the SPA and an XML type for the ESB, (in effect a small [[Anti-Corruption Layer]], my reading rather than a stated design).
- **Legacy.** No DTOs: WebWork `ModelDriven` form beans for input (`TGMDPVENc002.java:47-52`, `TGMDPVENc002Form`) and hand-built `org.json.JSONObject` for calls to the ADS adapters (`:130`, response parsed at `:271-272`, `obj.get("RTN_STATUS")`). The `RTN_*` names in modern DTOs come from exactly this style.

## Why it matters
Most integration failures are a disagreement about the shape of a payload. When a field is "missing" in the UI, check the `@JsonProperty`/`@JsonAlias` first.

## Check yourself
Why is `NwisPatrolStartResponseDto.status` a `Boolean` serialised as `RTN_STATUS`, and what breaks if you rename the Java field only?

## Related
- [[DTO Pattern]]
- [[HTTP and REST]]
- [[Validation]]
- [[Anti-Corruption Layer]]
- [[tgmdp_backend Error Handling and Validation]]
- [[Vendor Portal API Layer]]
- [[Vendor Portal Feature Trace - TGMDPVENc003 Serial Number Query]]
- [[What an ESB Is]]
- [[WebWork Actions vs Spring Controllers]]
- [[Concepts MOC]]
- [[System Overview]]
