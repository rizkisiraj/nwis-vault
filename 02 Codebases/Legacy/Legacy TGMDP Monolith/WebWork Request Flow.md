---
tags: [legacy, tgmdp-legacy]
---

# WebWork Request Flow

> How one click in the legacy TGMDP console becomes an HTML page. Up: [[Legacy TGMDP Monolith]] · [[System Overview]]

The legacy console is **classic server-side [[MVC]] with a [[Front Controller]]**. All requests
ending in `.action` go through **one** dispatcher servlet. A config file decides which Java method
handles the request, a chain of interceptors prepares the request around that method, and the
method's return string picks a JSP that renders the full HTML page. The browser receives finished
HTML and nothing else.

We follow one real request: the **device query page** of module `TGMDPa002` (裝置查詢作業). The
user fills in the search form and presses 查詢 (Search).

## The whole trip in one picture

```mermaid
sequenceDiagram
    autonumber
    participant B as Browser (JSP page + jQuery)
    participant F1 as IPFilter
    participant F2 as FrontControllerFilter
    participant D as ServletDispatcher (WebWork)
    participant X as XWork config (xwork.xml + WW_XML/TGMDPa.xml)
    participant I as Interceptor stack (defaultStack)
    participant A as Action TGMDPa002
    participant M as DeviceModel + DBDAO
    participant DB as DB2
    participant V as Template01.jsp + TGMDPa002Q1.jsp

    B->>F1: POST /TGMDP/TGMDPa002/browse.action (form1 fields)
    F1->>F1: client IP on allow-list?
    F1->>F2: chain.doFilter
    F2->>F2: UserBean in HttpSession? ButtonBean has TGMDPa002_browse?
    F2->>D: chain.doFilter
    D->>X: ActionMapper: namespace /TGMDPa002, name browse
    X-->>D: class TGMDPa002, method browse, results
    D->>I: build ActionInvocation
    I->>A: model-driven pushes TGMDPa002Form; params copies request params into it
    A->>M: dao.open(); PageBreak → devicemodel.browseMAC(...)
    M->>DB: SELECT ... ROW_NUMBER() OVER (...) WHERE ...
    DB-->>M: rows
    M-->>A: ArrayList of Map rows
    A->>A: request.setAttribute("view_data", rows)
    A-->>I: return "success"
    I-->>D: result "success" is type="chain" → run action "begin"
    D->>A: TGMDPa002.begin() loads drop-down data
    A-->>D: "success" → /common/Template01.jsp?page=TGMDPa002/TGMDPa002Q1
    D->>V: forward
    V-->>B: full HTML page (form + result table)
```

The rest of this note goes through each box.

## 1. The browser posts a form to `*.action`

Every page has one big `<form id="form1">`, declared in the layout JSP
(`legacy/TGMDP/TGMDP/common/Template01.jsp:80`). Buttons do not call an API. They set the form's
`action` attribute and submit the whole form:

```js
// legacy/TGMDP/TGMDP/js/submitForm.js:24-32
$('#form1').attr("target",target);
if (encoding == "multipart/form-data") {
  $('#form1').attr("action",cmd );
} else {
  $('#form1').attr("action",cmd + '.action');
}
$('#form1').submit();
```

The query page validates dates in jQuery and then calls `submitForm('browse','1')`
(`legacy/TGMDP/TGMDP/TGMDPa002/TGMDPa002Q1.jsp:41-121`). The URL is relative, so from
`/TGMDP/TGMDPa002/…` it becomes `/TGMDP/TGMDPa002/browse.action`. **The URL path *is* the routing
key**: folder = module namespace, file name = action name.

## 2. Servlet filters run first

`web.xml` registers two filters and the WebWork servlet:

```xml
<!-- legacy/TGMDP/TGMDP/WEB-INF/web.xml:16-27 -->
<filter-mapping><filter-name>IPFilter</filter-name><url-pattern>/*</url-pattern></filter-mapping>
<filter-mapping><filter-name>FrontcontrollerFilter</filter-name><url-pattern>*.action</url-pattern></filter-mapping>
<servlet>
  <servlet-name>webwork</servlet-name>
  <!-- <servlet-class>com.opensymphony.webwork.dispatcher.ServletDispatcher</servlet-class> -->
  <servlet-class>com.sys.servlet.ServletDispatcher</servlet-class>
```

- `IPFilter` (`legacy/TGMDP/src/com/sys/filter/IPFilter.java`) lets the request through only if
  the client IP matches an allow-list, except for `/APP_UPDATE`.
- `FrontControllerFilter` (`legacy/TGMDP/src/com/sys/filter/FrontControllerFilter.java:19-164`)
  splits the URI into `task_id` (`TGMDPa002`) and `action` (`browse`), sends you to `Login.jsp` if
  there is no `UserBean` in the session, and calls `noPower` if your `ButtonBean` does not contain
  `TGMDPa002_browse`. Details: [[FrontControllerFilter and ButtonBean Permissions]].

Servlet filters are the [[Interceptor and Filter Chain]] pattern at the container level. Each one
either calls `chain.doFilter(...)` to pass the request on or writes a response itself and stops.

> The name "FrontControllerFilter" is a little misleading. The actual Front Controller (one entry
> point that dispatches to handlers) is the WebWork `ServletDispatcher`. The filter is a security
> gate placed in front of it.

## 3. The dispatcher finds the action in XWork config

`com.sys.servlet.ServletDispatcher` is a small fork of WebWork's own dispatcher. The team copied it
to turn off WebWork's built-in multipart wrapping because it broke their own upload code
(`legacy/TGMDP/src/com/sys/servlet/ServletDispatcher.java:58-70`, comment dated 2009/07/31). It then
asks the ActionMapper for a mapping and runs it:

```java
// legacy/TGMDP/src/com/sys/servlet/ServletDispatcher.java:80-90
ActionMapping mapping = ActionMapperFactory.getMapper().getMapping(request);
if (mapping == null) {
    response.sendError(404);
    return;
}
du.serviceAction(request, response, getServletContext(), mapping);
```

The mapping comes from XML. `src/xwork.xml` includes one file per module letter:

```xml
<!-- legacy/TGMDP/src/xwork.xml:4-7 -->
<xwork>
  <include file="webwork-default.xml" />
  <include file="/WW_XML/TGMDPa.xml" />
  <include file="/WW_XML/TGMDPb.xml" />
```

and the module file declares a **package** per task code, with a namespace, actions, and results:

```xml
<!-- legacy/TGMDP/src/WW_XML/TGMDPa.xml:19-34 -->
<package name="TGMDPa002" extends="webwork-default" namespace="/TGMDPa002">
  <global-results>
    <result name="error">/common/Template01.jsp?page=common/Result</result>
  </global-results>
  <action name="begin" class="com.prj.actions.tgmdpa.TGMDPa002" method="begin">
    <result name="success">/common/Template01.jsp?page=TGMDPa002/TGMDPa002Q1</result>
  </action>
  <action name="browse" class="com.prj.actions.tgmdpa.TGMDPa002" method="browse">
    <result name="success" type="chain">begin</result>
  </action>
  <action name="export" class="com.prj.actions.tgmdpa.TGMDPa002" method="export">
    <result name="success">/common/TemplateR4.jsp</result>
  </action>
  <action name="select" class="com.prj.actions.tgmdpa.TGMDPa002" method="select">
    <result name="success">/common/Template01.jsp?page=TGMDPa002/TGMDPa002U1</result>
```

Things to notice:

- **One class, many entry points.** `begin`, `browse`, `export`, `select`, `update`, `goInsert`,
  `insert`, `delete`, `detail`, `sendSMS` all map to methods on the same class (lines 23-56).
- **Results are names, not views.** The method returns `"success"` or `"error"`, and the XML maps
  that string to a JSP. This is the Controller → View indirection of [[MVC]].
- **`type="chain"`** runs another action on the server inside the same request. `browse` runs the
  search, then chains to `begin` to load the drop-down data and render the page. That is how one
  JSP shows both the form and the results.
- **`global-results`** sends every `"error"` to one shared error page, `common/Result.jsp`.

## 4. The interceptor stack wraps the action

None of the module packages declare interceptors. They all `extends="webwork-default"`, so every
action gets WebWork's `defaultStack`. From `webwork-default.xml` inside
`legacy/TGMDP/TGMDP/WEB-INF/lib/webwork-2.2.6.jar` (lines 161-180):

```xml
<interceptor-stack name="defaultStack">
    <interceptor-ref name="exception"/>
    <interceptor-ref name="alias"/>
    <interceptor-ref name="servlet-config"/>
    <interceptor-ref name="prepare"/>
    <interceptor-ref name="i18n"/>
    <interceptor-ref name="chain"/>
    <interceptor-ref name="debugging"/>
    <interceptor-ref name="model-driven"/>
    <interceptor-ref name="fileUpload"/>
    <interceptor-ref name="static-params"/>
    <interceptor-ref name="params"/>
    <interceptor-ref name="conversionError"/>
    <interceptor-ref name="validation">…</interceptor-ref>
    <interceptor-ref name="workflow">…</interceptor-ref>
</interceptor-stack>
```

The two that matter most here:

- **`model-driven`**: because `TGMDPa002 implements ModelDriven` and `getModel()` returns a
  `TGMDPa002Form`, the form bean is pushed onto the OGNL value stack.
- **`params`**: copies every HTTP parameter onto the value stack, so `NODE_ID1=...` ends up in
  `form.setNODE_ID1(...)`. This is the ancestor of Spring's `@ModelAttribute` /
  `@ParameterObject` binding.

An interceptor stack is a nested chain. Each interceptor runs code before the action, calls
`invocation.invoke()`, then runs code after. This is the same [[Interceptor and Filter Chain]] idea
as servlet filters, but inside the framework.

## 5. The Action does the work

```java
// legacy/TGMDP/src/com/prj/actions/tgmdpa/TGMDPa002.java:48-56
public class TGMDPa002 implements Action, ModelDriven {
    private TGMDPa002Form form = new TGMDPa002Form();
    public Object getModel() { return form; }
    public String execute() { return "success"; }
```

`browse()` (lines 130-197) follows the same steps you will find in hundreds of legacy methods:

1. Get the `HttpServletRequest` and session from the thread-local `ServletActionContext`.
2. Create a `DBDAO` through `DAOFactory`, then `dao.open()`.
3. Build a `Map` of criteria from the form, adding row-level security by hand (a level-2 user is
   forced to their own `NODE_ID2`, a level-3 user to their own `USER_ID`: lines 146-156).
4. Hand the model to `PageBreak`, which runs `DeviceModel.browseMAC` with paging.
5. `request.setAttribute("view_data", rows)`: put the result where the JSP will find it.
6. In `catch`: `dao.rollback()` and `ExceptionResolver.handle(...)`, then return `"error"`.
7. In `finally`: `dao.close()`.

A WebWork action is **instantiated per request** (unlike a Spring singleton controller), so storing
the `form` in a field is safe. The action talks straight to `HttpServletRequest`, though, which
makes it hard to unit-test. See [[WebWork Actions vs Spring Controllers]].

## 6. The JSP renders the view

The result path `/common/Template01.jsp?page=TGMDPa002/TGMDPa002Q1` points to a **layout JSP**.
It reads `page` and includes the module page inside a shared header, menu and footer:

```jsp
<%-- legacy/TGMDP/TGMDP/common/Template01.jsp:20-31, 115 --%>
if(!taskName.equals("") && request.getParameter("page") != null){
    page_id = StringUtility.replaceChars(request.getParameter("page"));
    includeJSP = "../" + page_id + ".jsp";
}
...
<jsp:include page="<%=includeJSP%>" flush="false" />
```

The module JSP takes the data back out of request scope with **scriptlets** (Java inside `<% %>`)
and loops over it to print table rows:

```jsp
<%-- legacy/TGMDP/TGMDP/TGMDPa002/TGMDPa002Q1.jsp:13-15, 327-328 --%>
UserID userID = (UserID) session.getAttribute("UserBean");
ArrayList SYS_LIST=(ArrayList)request.getAttribute("SYS_LIST");
ArrayList alView=(ArrayList)request.getAttribute("view_data");
...
for(int i = 0; i < alView.size(); i++){
    Map hm = (Map)alView.get(i);
```

Output is HTML-escaped by hand: every cell calls `StringUtility.escapeHtml(...)`. Forget one call
and you have an XSS hole. React escapes text by default, which is one of the gains discussed in
[[Server-rendered JSP vs React SPA]].

## Why this design made sense (in 2009)

- **No JavaScript platform yet.** Browsers were slow and inconsistent, so rendering on the server
  was the safe choice. The browser only had to display HTML and submit forms.
- **One deployable.** UI, logic and SQL ship together in one WAR. Deployment is "copy the WAR to
  Tomcat."
- **Session state is easy.** The user, their permissions and even search criteria (`whereParam`)
  live in the server session or hidden fields. See [[Stateless vs Stateful]].
- **Config over code for routing.** Adding a page means adding XML, which non-specialists could do.

## What it costs

- Every interaction is a **full page reload**. That is slow and loses scroll position and
  client-side state.
- The JSP is coupled to `request` attribute names (`"view_data"`) that are just strings. Rename on
  one side and the page silently shows nothing. No compiler checks it ([[Type Systems]]).
- The controller and view cannot be used by a mobile app. The mobile apps needed a separate set of
  backends ([[Legacy ADS Adapter Backends]]). A JSON API serves both.
- Testing needs a running Tomcat and DB2. There are no unit tests in the repo.

## Try it yourself

1. Open `legacy/TGMDP/src/WW_XML/TGMDPa.xml` and find `TGMDPa003`. What URL does its `update`
   action answer, and what page is shown after it succeeds?
2. In `TGMDPa002.java`, find `export()` (line 198). Its result is `/common/TemplateR4.jsp`, not
   `Template01.jsp`. Open that JSP and work out why export needs a different template.
3. Trace `submitForm('select', …)` from `TGMDPa002Q1.jsp` to the JSP it renders.

## Related

- Up: [[Legacy TGMDP Monolith]] · [[System Overview]]
- Sideways: [[FrontControllerFilter and ButtonBean Permissions]] · [[Legacy Data Access with DBDAO]]
- Modern comparison: [[WebWork Actions vs Spring Controllers]] · [[Server-rendered JSP vs React SPA]]
- Concepts: [[MVC]] · [[Front Controller]] · [[Interceptor and Filter Chain]] · [[Server-Side Rendering]] · [[HTTP and REST]] · [[Routing]] · [[Authentication and Sessions]] · [[Stateless vs Stateful]] · [[Error Handling]] · [[Validation]] · [[Type Systems]] · [[Client-Server Architecture]]
