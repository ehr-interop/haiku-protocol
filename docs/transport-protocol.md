# Epic Haiku 11.9.2 (Android) — Wire Protocol Documentation

All class and field names below are as they appear in the shipped client
(short single-letter package names such as `t60`, `c10`, `q60` are the client's own
internal names, reproduced verbatim because they identify the classes unambiguously).

Key classes:
- `com/epic/communication/webservices/execution/CommandWebServiceOperation` — abstract HTTP transport
- `com/epic/communication/webservices/execution/CommandWebAPIServiceOperation` — JSON Web API transport
- `com/epic/haiku/communication/internal/BaseFacadeServiceOperation` — SOAP transport
- `com/epic/haiku/communication/internal/facade/FacadeServiceOperation`, `facade/a-f` — SOAP body serializer + fault parser
- `t60/a` (base Haiku ClientInfo), `t60/i`, `t60/e` (Haiku WebAPI client info), `com/epic/login/oauth2/OAuth2ClientInfo`
- `c10/c` (per-command ExecutionInfo), `c10/d` (base client info), `q60/f` (facade execution info), `q60/d` (facade command)
- `com/epic/communication/Connection`, `com/epic/communication/d` (Command)

---

## 1. Big picture: two transports over the same HTTP base

Haiku ships **two** ECF ("Epic Communication Framework") transports, both plain
`java.net.HttpURLConnection` `POST`s built by the same abstract base class:

| | SOAP facade (legacy) | Web API (JSON, default) |
|---|---|---|
| Operation class | `com.epic.haiku.communication.internal.facade.FacadeServiceOperation` (via abstract `BaseFacadeServiceOperation`) | `com.epic.communication.webservices.execution.CommandWebAPIServiceOperation` |
| Content-Type (req) | `application/soap+xml; charset=utf-8` | `application/json; charset=utf-8` or `multipart/mixed+json; boundary=<uuid>` |
| Accept (resp) | `application/soap+xml` | `application/json` or `multipart/mixed+json` |
| Relative path | `httplistener.ashx` (single dispatcher) | `internal/<seg1>/<seg2>/<CommandName>` (`getRequestRelativePath`), or a per-command override |
| Command addressing | Operation element in SOAP Body: name = last dot-segment of command URN, xmlns = URN minus last segment | Command name is the last URL path segment (e.g. `internal/Mobile/Internal/Epic.Mobile.Core.SearchRecords`) |
| Command object | `q60/d` (carries `getOperationClass` == `FacadeServiceOperation`) | `com.epic.communication/d` (generic Command) |
| Body payload | Reflection XML serializer `facade/b` + `facade/c` | Reflection JSON serializer `b10/d.C(Object)` over ECF metadata `a10/a` |
| Transport layer tag (perf errors) | `InterconnectFacade` | `MobileECF-WebAPI` |

### Dispatch decision (Connection.operationForCommand)

```
if (command.getOperationClass != null) → instantiate that class (q60.d ⇒ FacadeServiceOperation = SOAP)
else if (connection.getClientInfo != null) → clientInfo.m(command) ⇒ CommandWebAPIServiceOperation (JSON)
```

So:
- **Modern JSON**: every `com.epic.communication.d` command named `Epic.Mobile.Core.*` — e.g. `Epic.Mobile.Core.PreLogin`, `Epic.Mobile.Core.Authentication.Login`, `Epic.Mobile.Core.Authentication.LoginWizard` (ECFLoginWizard), `Epic.Mobile.Core.Authentication.EpicOAuth2Login`, `Epic.Mobile.Core.CreateSession`, `Epic.Mobile.Core.KeepAlive`, `Epic.Mobile.Core.Logout`, `Epic.Mobile.Core.SearchRecords`, `Epic.Mobile.Core.GetPatientChecks`, `Epic.Mobile.Core.TerminateRemoteSession`, `Epic.Mobile.Core.SaveUserSettings`, `Epic.Mobile.Core.UploadLog`, `Epic.Mobile.Core.TrackFeatureUseMulti`, etc. (registry in `t60/i.<clinit>`, xref.
- **Legacy SOAP**: every `q60.d` command whose name is a full EpicOnHand URN, e.g. `urn:epicsystems-com:EpicOnHand.2014.Services.Rover.MAR.RNVerify` (150 such URNs in the client string table; MAR/Rover, Lab, Flowsheets, InBasket, Login, PatientManager, OrderEntry, …). There is **no `SOAPAction` HTTP header and no `wsa:Action`** — the action is carried by the Body operation element's name + default namespace (see §3). The only `SOAPAction` string in the app is an unused library constant.

`t60/a.m(command)` (JSON side) additionally refuses to run any command when
`serverInfo.versionIsAtLeast(executionInfo.u)` fails (default minimum `8.1`, set in `t60/a.f`), producing a local
`CommandUnsupportedOperation` instead of a network call.

---

## 2. URL construction

`CommandWebServiceOperation.getEndpointURL`:

```java
String override = executionInfo.g; // c10.c field f, set by ClientInfo.f(command) via o(...)
String relPath = (override != null) ? override: this.getRequestRelativePath;
return new URL(executionInfo.d /* c10.c field b = base URL */, relPath);
```

- **Base URL** = environment Address, normalized in `e10.g.r`: trailing
 `/httplistener.ashx` is stripped, a trailing `/` is appended if missing. i.e. the environment address
 configured per-server in the environment list (typically
 `https://<server>/<Interconnect-VirtualDir>/`).
- `new URL(base, relPath)` — relative paths resolve with the standard java.net rules against the base dir.

### 2.1 SOAP relative path
`BaseFacadeServiceOperation.getRequestRelativePath` → constant **`httplistener.ashx`**
(→ `https://<base>/httplistener.ashx`). Same constant is used by `t60.a.s` which pre-builds
`new URL(env, "httplistener.ashx")`.

### 2.2 Web API relative paths
`CommandWebAPIServiceOperation.getRequestRelativePath`:

```java
String.format("internal/%s/%s/%s", executionInfo.e, executionInfo.f, command.getName)
```

`executionInfo` (`c10.c`) is populated by `c10.d.r` from the client-info constructor args of `t60.a`:

```
super(envURL, "Mobile", "Internal", TimeZone.getDefault, 30.0 /*default cmd timeout*/, …)
```

- `e` → field d = **`"Mobile"`** (2nd ctor arg)
- `f` → field e = **`"Internal"`** (3rd ctor arg), but `t60.a.f(command)` overwrites it with
 **`"Unauthenticated"`** for commands with `getRequiresAuthentication == false` (all login/pre-login commands).

So the default Web API paths are:

```
https://<base>/internal/Mobile/Internal/Epic.Mobile.Core.SearchRecords (authenticated)
https://<base>/internal/Mobile/Unauthenticated/Epic.Mobile.Core.Authentication.Login (pre-login, UNCERTAIN — see §8)
```

**Per-command path overrides** (ClientInfo `f(command)` calls `c10.c.o(path)`, which also forces the
JSON transport in `t60.a.m`):

| Command name | Override path | Reference |
|---|---|---|
| `Epic.Mobile.Core.Authentication.EpicOAuth2Login` | `internal/Auth/OAuth2/EPICOAUTH2LOGIN` | `t60/i.f`, |
| `Epic.Mobile.Core.CreateSession` | `api/epic/2017/Auth/OAuth2/CREATESESSION` | `OAuth2ClientInfo.f` and `t60/i.f` |

These two paths (plus `httplistener.ashx`) are the **only** literal paths in the app — all other Web API
calls use the constructed `internal/%s/%s/%s` form (unverifiable directly, but the string
`internal/Auth/OAuth2/EPICOAUTH2LOGIN` has exactly one code reference — the override table in `t60/i`).

---

## 3. SOAP transport (facade commands)

Source:.

### 3.1 HTTP
- `POST`, Content-Type `application/soap+xml; charset=utf-8`, Accept `application/soap+xml`.
- Path `httplistener.ashx` (§2.1).

### 3.2 Request envelope (reconstructed from `serializeRequestToStream` + `writeSOAPEnvelopeHeader`)

```
POST https://<base>/httplistener.ashx
Content-Type: application/soap+xml; charset=utf-8
Accept: application/soap+xml
Interconnect_SessionToken: <session token GUID> (only if command.requiresAuthentication)
Interconnect_ClientTime: Sat, 11 Nov 2026 18:00:00 GMT (com.epic.util.u.a, RFC1123, GMT)
Epic-External-Request: true (only when client-config flag c10.a.e is set)
X-Forwarded-For: <vb0.d.a.k address> (only when non-empty)

<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
 <soap:Header>
 <Interconnect_ClientTime xmlns="urn:epicsystems-com:Interconnect.Headers">Sat, 11 Nov 2026 18:00:00 GMT</Interconnect_ClientTime>
 <!-- optional, only when a debug id exists (c10.a.d): -->
 <Epic-Debug-ID xmlns="urn:epicsystems-com:Interconnect.Headers">…</Epic-Debug-ID>
 <!-- optional, only for Rover client against server < 8.2: -->
 <ClientTimeZone xmlns="urn:epicsystems-com:Interconnect.Headers"
 xmlns:ic="urn:epicsystems-com:Interconnect.Headers"
 ic:format="OLSON" ic:name="America/New_York"/>
 </soap:Header>
 <soap:Body>
 <RNVerify xmlns="urn:epicsystems-com:EpicOnHand.2014.Services.Rover.MAR"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 Guid="<session token>">
 <MARRNVerifyRequest> <!-- only when executionInfo.s != null -->
 … property elements from facade serializer …
 </MARRNVerifyRequest>
 </RNVerify>
 </soap:Body>
</soap:Envelope>
```

Details:
- Body operation element name/namespace come from `q60.f` execution info, filled in `t60.a.f(command)`
 for `q60.d` commands: `element = <urn after last '.'>` (e.g. `RNVerify`),
 `namespace = <full urn minus last segment>` (e.g. `urn:epicsystems-com:EpicOnHand.2014.Services.Rover.MAR`).
- `Guid` attribute = the session token string (`c10.c.h`, same value as the `Interconnect_SessionToken`
 HTTP header, sourced from `EpicSettings.getSessionToken` in `c10.d.r`); present only when the
 command requires authentication. NOTE: it is *not* namespaced.
- The inner wrapper element (e.g. `<MARRNVerifyRequest>`) exists when the service explicitly set
 `q60.f.x(wrapperName)` (e.g. `RnVerify.a` does `q60.f.v(cmd).x("MARRNVerifyRequest")`).
 Commands that do not set it have the property elements written directly under the operation element.
- Property element names: from the `@transportName` member annotation (`q60.c`) when present, else the
 property name from reflection metadata (`ie0.a.g`): getter `getDualUserIDExt` → property name
 `DualUserIDExt` (the "get" prefix is cut, the first letter is **not** decapitalized). Same for
 `isX` → `X`. Class-level record name likewise from `@transportName` (`q60.b`) or the simple class name.
 (Cf. §5/§8 for a caveat on casing.)
- Value encoding (`facade/b.c` writer): String → text; Boolean/Short/Int/Long/Float/Double → text;
 `java.util.Date` → `r60.a.c(text)` on read (ISO-like ECF date); `byte[]` → `p3.n0` (base64 per
 le0 XML layer); enum (`com.epic.util.z`) → `.getValue`; `Map`/`Collection`/array → repeated child
 elements (name = property name, one per item); nested ECF objects → nested element named per
 `facade.c` metadata; `xsi:nil="true"` handled on read.

### 3.3 Response

`deserializeResponseFromStream`:
1. Parse `<soap:Envelope>`, skip `<soap:Header>` if present.
2. Next child of `soap:Body`: if it is `soap:Fault` → fault processing (§3.4).
3. Otherwise `readSOAPResponse(reader)` = `facade/b.b(reader, responseObject)`:
 - first reads the operation element's **attributes** into matching properties
 (`v0_1.a(p9.d(i))` — keyed by attribute name), then
 - walks child elements, matching each name to a response property, recursively deserializing
 (nested objects, lists, maps, byte[] via `le0.f.n0`).
 - The response element's *own name is not checked* — only its children matter, so the server's
 `<RNVerifyResponse …>`-style wrapper is tolerated.

### 3.4 SOAP fault

`facade/a` parser: extracts
- `faultcode` (text),
- `faultstring` (text),
- `detail` → first `<applicationfault>` or `<internalfault>` child, reading its text and its `type` attribute.

Error classification (`BaseFacadeServiceOperation.finishWithFault`) maps:
- `faultstring` containing `INVALID-CLIENT-TIME:` → `InvalidClientTime` error code
- `faultstring` containing `PATIENT-WORKSPACE-CLOSED` → `PatientWorkspaceClosed`
- `type == "FacadeServiceNotFound"` → `CommandNotFound` (404-equivalent)
- type `AspNetUnexpectedException` + `System.Web.HttpException` in faultstring → `RequestLengthExceeded`
- fault type `InvalidDSTDateTime` / time-zone fault types (`INVALID-OLSON-TIME-ZONE`,
 `OLSON-TIME-ZONE-NOT-MAPPED`, `MS-TIME-ZONE-NOT-MAPPED`, `UNRECOGNIZED-TIME-ZONE-FORMAT`, …) →
 `InvalidTimeZone` / `InvalidSerializedTime`
- HTTP 404 → `BadClientConfiguration` ("not pointing at a functioning Interconnect").
All faults produce HTTP-level `CommandErrorReport` with source type `FACADE`.

Reconstructed fault example (inferred — field order/logic from `facade/a`; not byte-exact):

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
 <soap:Body>
 <soap:Fault>
 <faultcode>soap:Server</faultcode>
 <faultstring>INVALID-CLIENT-TIME: …</faultstring>
 <detail>
 <applicationfault type="AspNetUnexpectedException">System.Web.HttpException …</applicationfault>
 </detail>
 </soap:Fault>
 </soap:Body>
</soap:Envelope>
```

---

## 4. JSON Web API transport (default)

Source:.

### 4.1 Request

- `POST` to the URL from §2.2.
- Plain (non-multipart) request: body = **flat JSON object of the request DTO's properties**
 (`b10/d.C(Object)` writes `{ "prop": value, … }`; there is *no* envelope, *no* command-name key —
 the command is identified by the URL path). Keys = ECF property transport names (§5).
 Content-Type `application/json; charset=utf-8`.
- Multipart request (used when the request DTO, or any nested object property, has a `Data`/`InputStream`
 property per `a10.a.a`): Content-Type `multipart/mixed+json; boundary=<UUID.randomUUID>`.
 Layout (byte-exact from `serializeMultipartRequestToStream`):

```
CRLF
--<boundary>CRLF
Content-Disposition: inline; name="0" ← only for parts with a name; the JSON part writes name="0"… see note
CRLF? — actually: JSON part passes name=null ⇒ no Content-Disposition line
Content-Type: application/json; charset=utf-8
CRLF
<flat JSON of request DTO>
CRLF
--<boundary>CRLF
Content-Disposition: inline; name="<propertyName>"
Content-Type: application/octet-stream
CRLF
<raw bytes of the InputStream/byte[] property>
CRLF
--<boundary>--
CRLF
```
 Note (from `writeContentStart`): the JSON first part is written with `name=null` → **no**
 `Content-Disposition`; each binary part carries `Content-Disposition: inline; name="<property name>"`.
 The boundary is a random UUID string (no quotes).

### 4.2 Response

- `Accept: application/json` (or `multipart/mixed+json`).
- Plain response: the response DTO deserialized directly from JSON (`d10/b` reader over `ge0` JSON
 parser; `y(responseObject)` fills properties, tolerant of `xsi:nil` — no, `xsi:nil` is irrelevant
 here; nulls handled as JSON null).
- Multipart response (`d10.a.d(stream)` chunk parser): first chunk **must** be JSON (deserialized into
 the response object); every following chunk has header key `content-disposition:name` (sic — parser
 looks up `"content-disposition:name"`, quotes stripped) and is stored on the response object by
 property name (`a10.g.f(response, name, bytes)`).

### 4.3 Error envelope

For any HTTP status ≥ 400, `readServerErrorResponse` buffers the body and parses it as JSON
(`ge0.b`):

- If the JSON is a **string** → used as the error message directly.
- If it is an **object** → parsed into `com.epic.communication.webservices.execution.l` with the keys
 (ASP.NET Web API convention, keys are exact):

```json
{
 "ExceptionType": "…",
 "Message": "…",
 "ExceptionMessage": "…",
 "StackTrace": "…",
 "InnerException": { … same shape, recursive … }
}
```

- Additionally, `onClientErrorStatus` inspects the **HTTP response reason phrase**
 (`HttpURLConnection.getResponseMessage`, `he0/c.n`) for sentinel codes:

| Reason phrase contains | Mapped error |
|---|---|
| `PATIENT-WORKSPACE-CLOSED` | PatientWorkspaceClosed |
| `PATIENT-MERGED` | PatientMerged |
| `SESSION-EXPIRED` | SessionExpired |
| `MOBILE-INTENTIONAL-THROW` | MobileTestServerError |
| `INVALID-CLIENT-TIME` | InvalidClientTime |

 and on HTTP 500 error objects, `Message` containing `INVALID-AUTH-TOKEN` → `NotAuthorized`,
 `INVALID-CLIENT-TIME` → `InvalidClientTime`.

- HTTP status mapping (400/401/404 default): 400 → `ServerCodeError`, 401 → `AuthorizationRequired`,
 404 → `CommandNotFound`, other 4xx → `BadRequest`, 5xx → `ServerFailure` (or fault-derived codes).
- The response header **`ServerMetrics`** (first value) is parsed into
 `com.epic.core.performance.ServerResponseInstrumentation` and attached to command metrics.
- Perf error reports use source type `WCF` (client-error) / `ECF` (server/deserialization).

### 4.4 Auth header injection (Web API)

Client-info hook `b(info, HttpURLConnection)` (`com.epic.communication.webservices.execution.n`):

- `t60/i.b`: for commands in `{Epic.Mobile.Core.Authentication.EpicOAuth2Login, Epic.Mobile.Core.CreateSession}`
 when environment setting `OAUTH2.MAINTAIN_SESSION` is false, adds
 `Authorization: <tokenType> <token>` (`v(type, token)` = `type + " " + token`; type is
 `s` = `t60/i.r/s` … stored via `v(token, type)`).
- `OAuth2ClientInfo.b`: same header for `Epic.Mobile.Core.CreateSession` (or every command when
 `OAUTH2.MAINTAIN_SESSION` is true).
- `t60/e.b`: adds `Authorization` whenever `t` is set and `environment.w` (`OAUTH2.MAINTAIN_SESSION`).

So after OAuth2 login, every subsequent Web API call carries a Bearer-style
`Authorization: <scheme> <accessToken>` header, *in addition to* `Interconnect_SessionToken`
(only for commands with `requiresAuthentication == true`).

---

## 5. ECF serialization metadata (JSON/property naming)

Two annotation families exist (same semantics):
- SOAP facade: `q60.b` (class, `transportName`), `q60.c` (member: `transportName`, `local`,
 `isDefaultProperty`, `useAlternateKeyType`).
- JSON Web API: `a10.c` (class, `name` — the "ECF record name", e.g. `Epic.Mobile.Core.CreateSession.Request`),
 `a10.d` (member: `transportName`, `local`), with member type enum `a10.e` = {`Data`, `InputStream`,
 `Object`, …}. A class is multipart when any member (recursively) is `Data`/`InputStream`.

Name derivation (`ie0.a.g`, `ie0/b.g`): public fields use the field name (incl. leading `_` if so
declared); getters/setters produce name = `getXxx`→`Xxx`, `setXxx`→`Xxx`, `isXxx`→`Xxx`
(**first letter kept uppercase**). `@transportName` overrides the name. The writer (`b10/d.C`)
emits the JSON key via `a10.f.f` (transport name, else derived name); the reader matches the same.

Date handling: ECF JSON reader parses dates with `com.epic.util.n.a` (ThreadLocal `DateFormat`);
the only plausible pattern in the app is `yyyy-MM-dd'T'hh:mm:ss.SSS'Z'`.
UNCERTAIN whether writer uses the identical pattern — the writer uses a separate ThreadLocal
(`d10.c$a`), same format assumed.

---

## 6. Representative messages

### 6.1 SOAP facade pair: `Rover.MAR.RNVerify`

Service entry (`com/epic/clinical/mar/services/RnVerify`, — note the wrapper name:

```java
q60.d cmd = new q60.d(conn, "urn:epicsystems-com:EpicOnHand.2014.Services.Rover.MAR.RNVerify",
 RnVerify$Request.class, RnVerify$Response.class);
((RnVerify$Request) cmd.getRequest).setOrders(orders); // ArrayList<Medication>
((RnVerify$Request) cmd.getRequest).setDualUserIDExt(userExt);
((RnVerify$Request) cmd.getRequest).setState(state);
q60.f.v(cmd).x("MARRNVerifyRequest"); // inner request element
cmd.beginExecuteWithEvents(listener);
```

Request DTO (`RnVerify$Request`): `orders` (`ArrayList`), `dualUserIDExt` (String), `state` (enum `State`).
Response DTO (`RnVerify$Response`): `verifyResult` (String).

Request on the wire (reconstructed):

```
POST https://haiku.example.org/httplistener.ashx
Content-Type: application/soap+xml; charset=utf-8
Accept: application/soap+xml
Interconnect_SessionToken: 2a1b…-GUID
Interconnect_ClientTime: Wed, 11 Nov 2026 14:00:00 GMT

<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
 <soap:Header>
 <Interconnect_ClientTime xmlns="urn:epicsystems-com:Interconnect.Headers">Wed, 11 Nov 2026 14:00:00 GMT</Interconnect_ClientTime>
 </soap:Header>
 <soap:Body>
 <RNVerify xmlns="urn:epicsystems-com:EpicOnHand.2014.Services.Rover.MAR"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" Guid="2a1b…-GUID">
 <MARRNVerifyRequest>
 <Orders><…Medication…></Orders>
 <DualUserIDExt>1234</DualUserIDExt>
 <State>Signed</State>
 </MARRNVerifyRequest>
 </RNVerify>
 </soap:Body>
</soap:Envelope>
```
(ELEMENT CASING UNCERTAIN — see §8.3.)

### 6.2 JSON Web API pair: `Epic.Mobile.Core.CreateSession`

- Path override `api/epic/2017/Auth/OAuth2/CREATESESSION` (§2.2), no auth, multipart=off.
- Request DTO (`com/epic/haiku/communication/session/ECFLogin$Request`, — flat JSON,
 all keys camelCase as declared in Kotlin:

```json
{
 "loginID": "jsmith",
 "loginIDType": 1,
 "clientTime": "2026-11-11T14:00:00.000Z",
 "clientSecret": "<shared client secret>",
 "encryptedPassword": "<base64 RSA(AES-PW)>",
 "encryptedAESKey": "<base64 RSA-PKCS#1(aesKey)>",
 "passwordIV": "<base64(iv)>",
 "systemInfo": { … "Epic.Mobile.Core.Authentication.SystemInfo" … },
 "applicationInfo": { … "Epic.Mobile.Core.Authentication.ApplicationInfo" … },
 "deviceInfo": { … },
 "localeInfo": { … },
 "clientTimeZone": { … },
 "clientConfiguration": { "disassociateOnLogout": false },
 "loginNavigationInfo": { … },
 "refreshToken": "…", // getter over _authToken
 "tokenType": "…",
 "openIDConnectCredential": { … } // OIDC path only
}
```
 Password crypto (`session/a.n`): when a server RSA public key is present → random AES key + IV,
 password encrypted with AES (`com.epic.util.a.e`), key & IV RSA-encrypted with the server key
 (`com.epic.util.io.l.b(pk,"RSA")`), all base64 (`com.epic.util.d.b`); legacy fallback =
 MD5 hex of the password (`com.epic.util.h.f.h`).
- Pre-login (`Epic.Mobile.Core.Authentication.EpicOAuth2Login` → `internal/Auth/OAuth2/EPICOAUTH2LOGIN`)
 uses `EpicOAuth2Login$Request {}` (empty) and returns
 `{ "oAuth2Info": { "loginToken": "…", … } }` (`EpicOAuth2Login$Response`).
- Keep-alive (`Epic.Mobile.Core.KeepAlive`) body = `KeepAliveRequest` {}; response `KeepAliveResponse`.
- Logout (`Epic.Mobile.Core.Logout`) body = `ECFLogoutRequest`.

### 6.3 JSON request DTO naming for clinical commands

`RnVerifyECF$Request` (Kotlin, is the Web API variant of §6.1:
`orders` (ArrayList), `dualSignEmpID` (String), `orderIDs` (List of order IDs).

---

## 7. Session, keep-alive, SSL

- **Session token**: `EpicSettings.getSessionToken` (set on login) → HTTP `Interconnect_SessionToken`
 on every authenticated command; `Guid` attribute inside SOAP bodies.
- **Blob token**: separate `Interconnect_WebBlobToken` header (used by `BlobDownloadRequest`)
 for media/blob downloads.
- **Keep-alive**: `Connection.beginKeepAlive` runs `Epic.Mobile.Core.KeepAlive` (JSON) or, for
 servers without `MOBILE_ECF_LOGOUT_CAPABILITY`, the URN command
 `urn:epicsystems-com:EpicOnHand.2010.Services.Login.KeepAlive` (SOAP facade).
 Same capability gate selects `Epic.Mobile.Core.Logout` vs
 `urn:epicsystems-com:EpicOnHand.2009-08.Services.Login.Logout` (legacy `LegacyLogoutRequest`).
- **Timeouts**: command default timeout **30** (`t60/a` ctor constant 4629137466983448576 = 30.0);
 socket read timeout = `command.getTimeout * 1000` ms (constant 4652007308841189376 = 1000.0).
 Child "sync" connections get a fixed session timeout of 300 (4643985272004935680).
- **SSL** (`com/epic/communication/webservices/execution/j`):
 wraps the platform factory, re-creating sockets with an SSLContext restricted to
 **TLSv1.2/TLSv1.3** (picks the first of `TLSv1.3`,`TLSv1.2` enabled by default config);
 a `HandshakeCompletedListener` fails the connection with `SSLException("<proto> not supported")`
 if the negotiated protocol is `SSL` or `TLSv1.1`. No certificate pinning, no custom truststore.
 If the base URL is not https and the build requires TLS (`vb0.d.a.o`), the command fails
 with `BadClientConfiguration`/`SSLException("Https is required")`.
- Connection-level header hooks: `RequestResponseProcessor` (`execution/n`) — `a(info, conn)` and
 `b(info, conn)` — inject the `Authorization` header pre-connect (client-info side, §4.4); a second
 hook `a(info, code, headers, body, ms)` handles responses. (Not every hook invocation site was
 traced; see `CommandWebServiceOperation$b.w/v`.)

---

## 8. Uncertainties / open items

1. **`internal/Mobile/{Internal|Unauthenticated}/…` path segments.** Derived with high confidence from
 `t60.a(envURL, "Mobile", "Internal", …)` → `c10.d` fields l/k → `c10.r`→`c10.c` fields d/e, and
 `String.format("internal/%s/%s/%s", e, f, name)`. The `f="Unauthenticated"` overwrite for
 pre-login commands (`t60.a.f`) is also explicit. What is *not* verifiable from strings alone is
 whether the server canonicalizes these; only the two override paths appear as literals.
2. **Property-name casing on the wire.** Reflection metadata (`ie0.a.g`) keeps the getter's leading
 capital (`getDualUserIDExt` → `DualUserIDExt`). For the SOAP facade this is plausible (matching
 ECF/WCF element names), but for the JSON Web API the server may expect the exact same casing, or
 a decapitalized variant; the JSON examples in §6 assume keys as produced by the metadata
 (capital-preserving) with the DTO's declared (usually lowerCamel in Kotlin) names coming from
 Kotlin getters. Needs one live capture to pin down.
3. **`Epic-Debug-ID` as an HTTP header vs a SOAP header block.** It is definitively a SOAP header
 element (`urn:epicsystems-com:Interconnect.Headers`). The JSON side references
 `executionInfo.c` in `connectionFromCommand`, and the `setRequestProperty` call for that value
 was not traced, so a same-named HTTP header
 `Epic-Debug-ID` is likely but unproven.
4. **Exact ECF date wire format.** `yyyy-MM-dd'T'hh:mm:ss.SSS'Z'` is the only ISO-ish date pattern in the
 client's string table; the JSON reader uses `com.epic.util.n.a`. One live capture would confirm.
5. **SOAP response element name.** The client never checks it (§3.3); server-side name assumed
 `<CommandName>Response`.
6. **Multipart "name=0" detail.** The JSON first part is written with `name=null` (no
 Content-Disposition) — the `0` in older notes is wrong; binary parts carry
 `Content-Disposition: inline; name="<propertyName>"`.
7. **`SpecialSoap` / `soap+xml` bare strings** appear in ECF name tables but no code path was found
 sending a `SOAPAction` header; treat SOAP 1.2 action-less style as confirmed.
8. **`httplistener.ashx` vs base**: `e10.g.r` tolerates the address being stored *with*
 `httplistener.ashx` (strips it), implying old environment records pointed straight at the SOAP
 listener; current builds always resolve relative paths against the base dir.

---

## 9. Resolved uncertainties (verification pass 2)

Findings below were confirmed against the client's raw instruction stream, including branch
polarity where the high-level view was ambiguous.

### 9.1 Web API path segments — resolved (§8 item 1)

`Lt60/a;`:

- `t60.a` ctor calls `super(envURL, "Mobile", "Internal", …)`. In `c10.d` those two literals become
 fields `l` = **"Mobile"** and `k` = **"Internal"**. `c10.d.r(execInfo)` copies them into the
 per-command execution info: `execInfo.m("Mobile")` → `c10.c` field `d`, `execInfo.n("Internal")` →
 `c10.c` field `e`. `c10.c.e` returns field `d`, `c10.c.f` returns field `e`.
- `t60.a.f(command)` (verified) — the only override site of segment 3 in the entire app
 (grep: the literal `Unauthenticated` appears at exactly one instruction):
 ```java
 execInfo = new q60.f(command); // then super.r fills base URL/token/segments
 execInfo.z("8.1"); // min client version
 if (command instanceof q60.d /* SOAP */) // split URN into element name + namespace
 { w(lastDotSegment); y(urnSansLastSegment); x(null); }
 else { x("Request"); }
 if (!command.getRequiresAuthentication)
 execInfo.n("Unauthenticated"); // overwrites c10.c field e, i.e. f
 ```
- `CommandWebAPIServiceOperation.getRequestRelativePath` (verified):
 `"internal/%s/%s/%s"` with args `execInfo.e` (="Mobile"), `execInfo.f` (="Internal" or
 "Unauthenticated"), `command.getName` **verbatim, unsplit, un-URL-encoded**. The dot-splitting
 in `t60.a.f` is only for the SOAP execution info; the Web API path never transforms the name.

So the only two non-overridden Web API path shapes are:
```
https://<base>/internal/Mobile/Internal/Epic.Mobile.Core.SearchRecords (any auth or default)
https://<base>/internal/Mobile/Unauthenticated/Epic.Mobile.Core.PreLogin (requiresAuthentication == false)
```
`command._requiresAuthentication` defaults to **true** in every `com.epic.communication.d` ctor
(chains to the no-arg ctor, which sets the field to 1). The only call sites that clear it:
- `ha0.q.a` → `Epic.Mobile.Core.PreLogin` (`setRequiresAuthentication(0)`) → really uses
 `internal/Mobile/Unauthenticated/Epic.Mobile.Core.PreLogin`.
- `com.epic.login.oauth2.OAuth2ClientInfo.d` → `setRequiresAuthentication(0)` for
 `Epic.Mobile.Core.Login` (OpenID Connect branch) → `…/Unauthenticated/Epic.Mobile.Core.Login`.
- `session/h.r` → `Epic.Mobile.Core.Authentication.EpicOAuth2Login` /
 `Epic.Mobile.Core.CreateSession` — but both are in the forced-WebAPI/override set, and their
 requests carry the explicit path overrides (`internal/Auth/OAuth2/EPICOAUTH2LOGIN`,
 `api/epic/2017/Auth/OAuth2/CREATESESSION`, see §2.2), so their "Unauthenticated" segment is
 never actually exercised.

Precedence (from `getEndpointURL`): per-command URL override `c10.c.g` (field `f`, set by
`t60/i.f` / `OAuth2ClientInfo.f` via `o`) **beats** the computed
`internal/Mobile/<seg>/<Name>` path. Everything else uses the computed path.

Still ambiguous: server-side canonicalization (case-sensitivity of `Mobile`/`Internal`,
whether `Unauthenticated` aliases `Internal`) — client sends exactly these strings.

### 9.2 ECF date serialization — resolved (§8 item 4)

**JSON Web API writer**: `d10.c.k(java.util.Date)` →
`this.c.t(((java.text.DateFormat) d10.c.e.get).format(date))` — written as a **JSON string**
(`ge0.d.t` = quoted, escaped UTF-8). The `ThreadLocal` factory `d10.c$a` (verified)
constructs `new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss'Z'", Locale.ROOT)` and sets its
time zone to `UTC` — the pattern literal contains no fractional-seconds component.

So the Web API wire date is `"2026-11-11T18:00:00Z"` — seconds precision only, uppercase
24-hour HH, literal `Z`, **no milliseconds**. The `yyyy-MM-dd'T'hh:mm:ss.SSS'Z'` string
seen in the client string table belongs to a different formatter and is **not** used by the ECF JSON
writer (nothing reaches `d10.c$k` except `d10.c$a`).

**SOAP/ECF XML side**: writer `le0/l.t(Date)` → a ThreadLocal `com.epic.util.n` whose
`format` is hand-rolled: `"%04d-%02d-%02dT%02d:%02d:%02d"` (UTC) or ±offset — again
seconds precision, no millis, `Z` when zero offset.

**Reader** (both transports converge): `d10.b.k` (JSON) and `le0.b.c→le0.f.s0` (SOAP)
both parse with `com.epic.util.n.a` — one lenient ISO-8601 matcher, verified in
`com/epic/util/n <clinit>`:
`^(\d{4})(?:-?(\d{2})(?:-?(\d{2}))?(?:[T\s](\d{2})(?::?(\d{2})(?::?(\d{2})(?:\.(\d{3}))?)?)?([Zz]|([+-]\d{2}):?(\d{2})?)?)?)?$`
— date, or optional `T`/space-separated time (minutes/seconds optional, optional 3-digit
fraction), optional `Z`/`z`/`±HH:mm`; absent offset ⇒ `TimeZone.getDefault`. Dates are never
epoch numbers on either transport. Round-trip note: writers emit only `…T…:…:…Z` (JSON) /
offset form (ECF XML); the fraction+timezone tolerance is input-only.

Remaining ambiguity: none material for the client side; fractional-second input tolerance on the
reader is the only fuzz.

### 9.3 SOAP response body element — resolved (§8 item 5)

`BaseFacadeServiceOperation.deserializeResponseFromStream` (instruction-level verified):
1. `readSOAPEnvelopeHeader`: `P0("soap:Envelope")` (assert qualified name `soap:Envelope`); if the
 next child is `Header` (local name via `c10.c`-style `g`), skip its subtree (`a0`).
2. `O0` = assert current node is Element, advance — moves onto the **first child of
 `soap:Body`**, whatever its name.
3. If that element's qualified name (`h`) equals `"soap:Fault"` → `facade/a` fault parser (§3.4).
4. Else `readSOAPResponse(reader)` = `facade/b.b(reader, responseDTO)`, which at that point is
 positioned **on** the response element (past its start tag):
 - reads its **attributes** (`c`/`d(i)`/`a(i)`) and maps each attribute name to a response-DTO
 property annotated `q60/c` with `useAlternateKeyType:true` (`facade/c.a(name)`, the *only*
 attribute mechanism);
 - then walks child elements in order: name → `facade/c.c(name)` property; unknown names are
 skipped (`a0`); the element's own name is **never compared** (already established in §3.3).

So the client contract is precisely: *"the first child element of `soap:Body` carries the response;
its children (and, rarely, its attributes) must be the DTO property elements."* The conventional
server element name is `<CommandName>Response` (client class names end in `Response`, request
elements are bare `<CommandName>` with `<XRequest>` wrappers), but this is a **convention, not a
client-enforced name** — a server using a different wrapper would yield an empty (all-skipped)
response rather than an error. What the client *does* enforce: `soap:Envelope`, and `soap:Fault`
detection on `soap:Body`'s first child.

**Guid/session-token echo — no such thing on the response** (§8): `Guid="<session token>"` is
written only as an **attribute of the outgoing request operation element**
(`serializeRequestToStream`: `v7_1.g("Guid", execInfo.h)`, `h` = the session token copied from
`EpicSettings.getSessionToken` by `c10.d.r`). The response-side token arrives inside the login
response DTO — `ECFLogin$Response.SessionInfo.Token` (`com.epic.haiku.model.ECFSessionInfo._token`)
and the embedded `EpicSettings` (`settings["SessionToken"]`) — and is consumed by the login
workflow (`didBecomeAuthenticated(EpicSettings)`), not by the SOAP parser. The attribute-mapping
facility is essentially unused: a full scan of the client for `q60/c` annotations with
`useAlternateKeyType:true` finds only 4 DTO classes (`mockidpauthenticator/saml/AuthnRequest`,
`…/Issuer`, `…/NameId`, `patientlist/legacy/model/Mapping`,
`patientsearch/command/FindPatient$Response`) — none named `Guid`/session-token.

### 9.4 Epic-Debug-ID — resolved (§8 item 3): sent both as HTTP header and SOAP header element

Two independent write sites, both fed from the same execution-info string `c10.c.c`
(field `g`, set by `c10.d.r` from `c10.a.f`):

1. **HTTP header — all transports** (shared base class `CommandWebServiceOperation.connectionFromCommand`,
 verified at instruction level; this call had previously gone unnoticed):
 it calls `execInfo.c` (the debug-id string getter) and, if that value is non-null
 and non-empty, passes it to `HttpURLConnection.setRequestProperty` under the header
 name `Epic-Debug-ID` — on JSON Web API **and** SOAP facade requests alike.
2. **SOAP header element** — `BaseFacadeServiceOperation.writeSOAPEnvelopeHeader`:
 `<Epic-Debug-ID xmlns="urn:epicsystems-com:Interconnect.Headers">…</Epic-Debug-ID>`, written
 only when the same string is non-empty (i.e. SOAP requests carry it twice: header once, HTTP
 once).

The value (`e50.e.q`, verified) is built when the client config id is non-empty
(`c10.a.d` = `e50.e.g`):
- id length ≥ 20: `<id>||<appName>|<platformName>|<sanitizedDeviceName>|`
- id length < 20: `<id left-padded to 20 chars with '-'>||<appName>|<platformName>|<sanitizedDeviceName>|<rawId>|`
- `<sanitizedDeviceName>` = device-name string filtered against an ASCII whitelist; empty string (no header)
 when the client id is empty.

Also note in the same header block (HTTP): `Epic-External-Request: true` iff `c10.a.e`
(`e50.e.c` → `e50.a.a`), `Interconnect_ClientTime` always (`com.epic.util.u.a`,
RFC1123), `X-Forwarded-For` iff `c10.a.l` non-empty, `Interconnect_SessionToken` iff
`command.getRequiresAuthentication`.

### 9.5 Capability gating / dispatch: SOAP-URN vs Web API — resolved (§5 of task, §8 item 8 context)

Dispatch is **per-command, not per-connection**, and there is no global "SOAP on/off" flag:

1. `Connection.operationForCommand(command)`:
 - `command.getOperationClass != null` → reify that operation. Because
 `q60.d.getOperationClass` is hard-wired to
 `com.epic.haiku.communication.internal.facade.FacadeServiceOperation` (verified),
 **any `q60.d` command is SOAP, unconditionally** — `clientInfo.m` is never consulted for it.
 - otherwise → `connection.getClientInfo.m(command)`.
2. `c10.d.m(command)` → always `new CommandWebAPIServiceOperation(command)`. `t60.a.m(command)`
 (verified branch polarity) is the real Web API gate:
 ```java
 if (forcedWebApiSet.contains(command.getName)) → WebAPI
 exec = command.getExecutionInfo /* q60.f, from clientInfo.f */;
 if (exec.g != null) /* per-command URL override (c10.c.f) */ → WebAPI (EpicOAuth2Login, CreateSession)
 minVer = exec.u; // "8.1" set by t60.a.f for every command
 if (minVer != null && serverInfo.versionIsAtLeast(minVer)) → WebAPI
 else → CommandUnsupportedOperation (no network call)
 ```
 So a *non-SOAP* command is Web API unless the server version pre-dates its per-command minimum
 (`q60.f.j`, default `8.1`; per-command overrides raise it via `z(...)` — the only other caller is
`com.epic.inbasket.patientcalls.e`, which sets `"8.2"` for the patient-call commands).
3. `ServerInfo` is the capability source:
 `_capabilities: HashSet<String>`, `_version: String` (e.g. `"8.4"`), built from the login
 response (`ECFLogin$Response.ServerInfo` → `ECFServerInfo`).
 `hasCapability(Capability c)` = `_capabilities.contains(c.getValue) || versionIsAtLeast(c.getVersion)`
 (either condition passing suffices; `Capability.getValue` string may be empty ⇒ capability
 reduces to the version test).
 The gating use is at command-construction time, not dispatch time
 (`t60.a.c/e`, i.e. keep-alive/logout):
 - `!serverInfo.hasCapability(HaikuCapability.MOBILE_ECF_LOGOUT_CAPABILITY)` (value string
 **"ECFLogout"**, no version — `HaikuCapability <clinit>`) → build `q60.d`
 legacy SOAP command with URN
 `urn:epicsystems-com:EpicOnHand.[2010|2009-08].Services.Login.[KeepAlive|Logout]`.
 - capability present → plain `com.epic.communication.d` Web API command named
 `Epic.Mobile.Core.KeepAlive` / `Epic.Mobile.Core.Logout`.
4. Per-client-info forced sets (constructor argument to `t60.a`):
 - `t60/i` (standard Web API client info): `{PreLogin, Authentication.Login, TerminateRemoteSession, SearchRecordsPreLogin}`;
 - `t60/e` (pre-login/OIDC client info): `{InitiateOpenIDConnectLogin, PreLogin, Login, Authentication.Login, TerminateRemoteSession}`;
 - `OAuth2ClientInfo`: `{CreateSession}` (+ path override).
 These bypass the version check entirely; all other Web API commands still pass through the 8.1 check.

Net rule of thumb: **q60.d ⇒ SOAP; everything else ⇒ JSON Web API, gated only by a minimum server
version (default 8.1) and the two per-command path overrides; keep-alive/log-out pick SOAP URNs
only when the server lacks the "ECFLogout" capability.**

