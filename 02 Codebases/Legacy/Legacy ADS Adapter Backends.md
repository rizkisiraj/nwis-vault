---
tags: [legacy, legacy-ads]
---

# Legacy ADS Adapter Backends

> The Java backends the legacy Cordova apps call: one servlet per system, one big `if/else` that dispatches by method name. Up: [[Legacy Systems]] · [[System Overview]]

The legacy mobile apps ([[niws2-app-legacy]], [[ams2-app-legacy]]) cannot use the JSP console,
because it returns HTML pages. They talk instead to a family of small Java web apps called
**ADS** (the code never spells the acronym out; read it as "adapter service"). Each ADS is a WAR
on Tomcat containing one **adapter servlet**. The app posts a method name and a JSON blob, and gets
JSON back.

| ADS | Servlet path | Used by | Typical adapters |
|---|---|---|---|
| `AMS_ADS` | `/AMS_ADS/ams/adapter` | AMS2 | `AMS_GetTicketList`, `AMS_UpdateTicket`, `AMS_PushAdapter`, `AMS_Photo` |
| `NWIS_ADS` | `/NWIS_ADS/nwis/adapter` | NWIS2 | `NWIS2_GetPatrolList`, `NWIS2_SavePatrol`, `NWIS2_PushAdapter` |
| `SD1_ADS` | `/SD1_ADS/sd1/adapter` | both | `SD1_GetSiteInfo`, `HIW_*`, `IRT_IRTUtility`, `SMR_*`, `TGMDB_ResSwapUtility` |
| `TGMDP_ADS` | `/TGMDP_ADS/tgmdp/adapter` | both + launcher | `TGMDP_AppAuth` (login/OTP), `TGMDP_CheckVer`, `TGMDP_VerifyMAC`, `TGMDP_MPLUSUtiltiy` |
| `ACPT_ADS` | `/acpt/adapter`, `/TGWebAPI/adapter`, `/VendorWebAPI/adapter`, CXF `/services/*` | acceptance app, web and vendor portal | `CtiModel` etc. |
| `MRM_ADS` | servlets + `webservice/MRM.java` | vendor portal, ESB | material requisition |

Copies live in `niws2-app-legacy/{NWIS_ADS,SD1_ADS,TGMDP_ADS,AMS_ADS}`,
`ams2-app-legacy/{AMS_ADS,SD1_ADS,TGMDP_ADS}` and `legacy/{ACPT_ADS,MRM_ADS}`. Each carries its own
copy of the `com.sys` framework described in [[Legacy Data Access with DBDAO]].

## The wire protocol

Every call is an HTTP **POST** with two form fields:

```
POST /AMS_ADS/ams/adapter
Content-Type: application/x-www-form-urlencoded; charset=UTF-8

invoke_method=AMS_GetTicketList_getTicketList
invoke_data={"SITE_ID":"N1234","ZONE":"N",...,"VERIFY_USER_ID":"...","VERIFY_UUID":"...","VERIFY_TOKEN":"..."}
```

- `invoke_method` is `<AdapterClass>_<method>`: a remote procedure name, not a resource URL.
- `invoke_data` is a JSON string carrying the parameters **plus three credentials** added by the
  client on every call (see `AdapterUtility.js` in [[Legacy Cordova App Anatomy]]).
- The answer is JSON with `RTN_STATUS` (true/false), `RTN_MESSAGE` and `RTN_DATA`. It is sent
  with HTTP **200 even when the call failed**. Errors are data, not status codes.

This is **RPC over HTTP**, a small home-made relative of SOAP. Compare with [[HTTP and REST]] and
[[SOAP vs REST]].

## Inside the adapter servlet

```mermaid
flowchart TD
  P[POST invoke_method + invoke_data] --> O[dao.open]
  O --> W{method on bypass list?<br/>CheckVer, SyncAllTicketData, CheckServices}
  W -- no --> V[LogModel.verifyLoginMst<br/>user_id + uuid + token vs A_LOGIN_MST]
  W -- yes --> OK
  V -- "" --> OK[authorised]
  V -- "token已過期 / 裝置資訊失效" --> E["{STATUS:0, MESSAGE:...}"]
  OK --> IF{"if/else on invoke_method<br/>~50 branches"}
  IF --> AD["new AMS_GetTicketList()<br/>setDao(dao)<br/>getTicketList(invoke_data)"]
  AD --> J[JSON string]
  J --> F[finally: out.println, dao.close]
  E --> F
```

The real code (`ams2-app-legacy/AMS_ADS/src/com/prj/servlet/ams/AdapterService.java`):

```java
// AdapterService.java:45-74 (trimmed)
String invoke_method = __request.getParameter("invoke_method");
dao.open();
Set<String> validValues = new HashSet<>();
validValues.add("AMS_CheckVer_checkVer");
validValues.add("SyncAllTicketData");
validValues.add("CheckServices");
if(!validValues.contains(invoke_method)){
    JSONObject jobj = new JSONObject(invoke_data);
    String user_id   = jobj.getString("VERIFY_USER_ID");
    String uuid      = jobj.getString("VERIFY_UUID");
    String old_token = jobj.getString("VERIFY_TOKEN");
    token_status = log_model.verifyLoginMst(user_id, uuid, old_token);
    isVerify = token_status.equals("");
}
```

```java
// AdapterService.java:226-233
else if(invoke_method.equals("AMS_GetTicketList_getTicketList")){
    String invoke_data = __request.getParameter("invoke_data");
    AMS_GetTicketList obj = new AMS_GetTicketList();
    obj.setDao(dao);
    rtn = obj.getTicketList(invoke_data);
}
```

Things to notice:

- **The dispatcher is a hand-written [[Front Controller]].** About 50 `else if` branches map strings
  to classes. Adding an endpoint means adding a branch. Spring's `@RequestMapping` scanning removes
  exactly this boilerplate.
- **Token check before dispatch** is a hand-rolled authentication filter
  ([[Authentication and Sessions]]). The token is checked against `A_LOGIN_MST` in DB2
  (`ams2-app-legacy/AMS_ADS/src/com/prj/models/LogModel.java:63-84`), so every call costs one extra
  query. The answer strings `token已過期。` (token expired) and `裝置資訊失效。` (device info invalid)
  are matched **by text** on the client.
- **Exceptions become an empty body.** The outer `catch` sets `rtn = ""` (lines 457-461). The client
  gets a 200 with nothing in it.
- **The same `dao` is passed to every adapter.** The servlet owns the connection and its
  `finally` closes it. Adapters that write call `commit()` themselves. Anything left uncommitted is
  rolled back on close.

## An adapter method

```java
// ams2-app-legacy/AMS_ADS/src/com/prj/adapters/ams/AMS_GetTicketList.java:29-54 (trimmed)
public String getTicketList(String __invoke_data) {
    AMSTicketModel model = new AMSTicketModel();
    model.setDao(dao);
    JSONObject jobj = new JSONObject(__invoke_data);
    Map<String, String> hmData = new LinkedHashMap();
    hmData.put("ZONE",    jobj.getString("ZONE").toUpperCase());
    hmData.put("SITE_ID", jobj.getString("SITE_ID").toUpperCase());
    ...
    PageBreak pb = new PageBreak();
    pb.setDao(dao);
    pb.setPageNumber(jobj.getInt("PAGE_NUMBER"));
    pb.setPageRecords(IniValue.getPAGE_RECORDS());
    pb.setModel(model, "selectTicketList");
    pb.doPageBreak(hmData);
```

It then copies each row map into a `JSONObject` field by field, cuts `MSG` to 30 characters, splits
rows into three lists by status, and returns
`{RTN_STATUS:true, RTN_DATA:{DATA_LIST1, DATA_LIST2, DATA_LIST3, TOTAL_PAGES, NOW_PAGES, ...}}`
(lines 56-131). An empty result returns `{RTN_STATUS:false, RTN_MESSAGE:"查無資料"}` (no data).
The adapter is part controller, part [[Service Layer]], part serializer, all in one method.

The README for this bundle records a real bug in the same file: a truly empty paged result
throws a NullPointerException on `al.size()` at line 65, so `查無資料` is never reached
(`ams2-app-legacy/README.md` §6).

## Talking to corporate systems: ESB SOAP

Some adapters do not stop at DB2. They call Taiwan Mobile's **ESB** (enterprise service bus) over
SOAP using JAX-WS stubs generated from WSDL:

```java
// ams2-app-legacy/AMS_ADS/src/com/prj/adapters/ams/AMS_UpdateTicket.java:151-156
ESBService service = new ESBService();
ESBPortType port = service.getPort();
...
_GetTicketOrginal_headerVal.setMessageTo("ESB");
```

Upstream systems behind the ESB (FTS, INAD, AMS fault system, SMR, IRT) are on the intranet. In
local development a fake ESB stands in for them (`tgmdp_fake_esb/` in the workspace, and the
"LOCAL DEV BYPASS" blocks in `AMS_AuthorityCheck.java`). See [[Message Integration and ESB]].

## Why this design

- **Mobile needed JSON, not HTML.** The console was server-rendered, so a separate data API had to
  exist for the apps. These adapters were it.
- **Mirrors IBM MobileFirst/Worklight "adapters".** The names `invokeProcedure` and
  `invoke_method` and the "adapter" naming look like the IBM Worklight hybrid-app platform, which
  used exactly this procedure-call model. That is my inference from the naming. Nothing in the
  repo states it.
- **Simple to deploy.** One servlet, one URL, firewall rules for a single path.

## What it costs

- One endpoint for everything means **no HTTP caching, no per-endpoint access logs, no
  per-endpoint auth rules**. Gateways and nginx cannot route on the method name. (The modern
  stack hit the reverse issue: staging nginx only routes `/api/`.)
- **Errors hidden in 200s** force every client to inspect bodies and match on strings.
- **No contract.** Field names exist only in Java `put("…")` calls and JS reads of `__data.X`.
  The rebuild has to recover them by reading both sides; that is what the parity process does.

## Questions to check understanding

1. Why can a proxy not tell a read from a write in this protocol?
2. What does the client see when an adapter throws a `NullPointerException`?
3. List the three credentials sent on every call. Where does each come from on the phone?

## Related

- Up: [[Legacy Systems]] · [[System Overview]]
- Used by: [[niws2-app-legacy]] · [[ams2-app-legacy]] · [[Legacy Cordova App Anatomy]]
- Built on: [[Legacy Data Access with DBDAO]]
- Modern comparison: [[Cordova+jQuery vs RN Shell + React Web Layer]] · [[WebWork Actions vs Spring Controllers]]
- Concepts: [[Front Controller]] · [[HTTP and REST]] · [[SOAP vs REST]] · [[Message Integration and ESB]] · [[Gateway]] · [[Authentication and Sessions]] · [[Error Handling]] · [[Serialization and DTOs]] · [[Service Layer]] · [[Client-Server Architecture]] · Remote Procedure Call
