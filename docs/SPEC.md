# Epic Haiku Wire Protocol — Consolidated Specification

**Target:** Epic Haiku 11.9.2 (`com.epic.haiku.android`) · Hospital Interconnect/Epic mobile web servers
**Status:** Complete for the documented client version. Only the items marked 🔴 in §9
require live observation to confirm; none affect building against the documented surface.
**Machine-usable companions:**
- `catalog/message-spec.json` — message catalog: 2,851 records, 15,963 members,
 wire names final (transportName overrides applied), request/response typed
- `catalog/capability-list.txt` — 388 `HaikuCapability` wire values
- Detail docs: `transport-protocol.md`, `auth-flow.md`, `org-discovery.md`, `security-webview-push.md`

---

## 1. Connection model

1. **Provisioning** — hospital IT delivers an encrypted configuration document ("econfig")
 out-of-band: deep link `epic://handheld/config/<payload>` (payload = AES/CBC ciphertext,
 key `"9FE3C86E96C5CC9C"`, IV `{0xD2,0xAD,0xBE,0xEF}×4` — same key as legacy
 "Epic Scramble") or MDM restriction key `EpicMobileConfigurationURL` (must be the
 encrypted `epic://` form; raw https links are rejected). Only other MDM key read:
 `restrictions_pending` (bool).
2. **econfig fetch** — `GET <econfig-url>` (30 s timeout) with `If-Modified-Since`/
 `If-None-Match` (304 = unchanged; ETag persisted). Container XML: `Guid`, `Name`,
 `Generated`, `Iterations`, `Salt`, `IV`, `PasswordHash`, `Data` (Base64 inner XML).
 - Mode A (no `PasswordHash`, default): PBKDF2WithHmacSHA1 (salt = Base64(Salt),
 iterations from `Iterations`, 256-bit key) → `AES/CBC/PKCS5Padding`; no password
 supplied → the raw `Salt` element text is used as the password.
 - Mode B (legacy "Epic Scramble", `PasswordHash` present): AES-128 with key from
 `SHA-1(UTF-16LE(password + "Epic Scrambler"))` hex-prefix, static IV
 `{0xD2,0xAD,0xBE,0xEF}×4`.
 - Cache: `<config-guid>.econfig` in app `filesDir` (private; legacy encrypted
 files migrated to plaintext in place).
3. **Configuration XML** (element-based, no attributes; root name not validated):
 `Configuration` → recursive `Group` (Guid/Name/Description + `<Settings><key>value`)
 / leaf `Environment` (+ `<Address>`, `<CustomSettings><Key>…<Value>…`).
 Pinned settings keys: `LOGIN.AUTH_TYPE` (1=OAUTH2, 2=OIDC), `OAUTH2.CLIENT_ID`
 (Epic-shipped IDs `80FFB3235CCD4B41891617711D958594` /
 `8B3A7DA58CF64F699BE96F9306F7A3FD`), `OAUTH2.MAINTAIN_SESSION`, `OIDC.ENABLE_SSO`,
 `LOGIN.DISABLE_REVEAL_PASSWORD`, `LOGIN.REMEMBER_LAST_USER`,
 `HEADER.DISASSOCIATE_ON_LOGOUT`, `DESKTOP.HYPERSPACE_URL`,
 `BARCODING.ALLOW_CAMERA_SCANNING`, `BARCODING.OVERRIDE_CAMERA_SCANNING_VENDOR/LICENSE_KEY`,
 `RequirePasswordForEdit` (Configuration level).
4. **Base URL rule** — `Address` minus trailing `/httplistener.ashx`; all commands
 POST to `<base>/<relative-path>`.
5. **Capability model** — `com.epic.communication.ServerInfo` fields: `encryptionKey`,
 `version` (benchmarks 7.8–10.6.0, `versionIsAtLeast`), `isAndroidSupported`,
 `capabilities` (Set of **wire-value strings**, not enum names — 401-constant
 `HaikuCapability` enum, 388 with wire values: full list in
 `capability-list.txt`), `readOnly`, `loginMessage`, `isUserLoggingEnabled`,
 `userHasEmbeddedHSAccess`, `envID`, `loginWizardEnabled`. Gate = capability
 present OR server version ≥ capability's version floor; missing capabilities fall
 back to legacy SOAP URNs.

## 2. Transports

### 2.1 JSON Web API (default for `Epic.Mobile.*` commands)

Path template: `internal/%s/%s/%s` = `internal/Mobile/<Internal|Unauthenticated>/<CommandName>`,
command name used **verbatim** (no splitting/transformation). The `Unauthenticated`
segment replaces `Internal` exactly when the command has `_requiresAuthentication=false`
— only `PreLogin` and `Epic.Mobile.Core.Login` exercise it. Per-command URL overrides
take precedence over the computed path:

```
POST <base>/internal/Mobile/Internal/<CommandName> (default, authenticated)
POST <base>/internal/Mobile/Unauthenticated/<CommandName> (PreLogin, Login)
POST <base>/internal/Auth/OAuth2/EPICOAUTH2LOGIN (EpicOAuth2Login override)
POST <base>/api/epic/2017/Auth/OAuth2/CREATESESSION (CreateSession override)
Content-Type: application/json; charset=utf-8
```

🔴 Server-side canonicalization (case sensitivity, `Unauthenticated` ≡ `Internal`)
only observable live.

**Transport dispatch per command** (client-side, deterministic):
1. Reified URN commands (`q60/d`) → **SOAP facade unconditionally**.
2. Forced-WebAPI sets (`PreLogin`, `Authentication.Login`, `TerminateRemoteSession`,
 `SearchRecordsPreLogin`, `InitiateOpenIDConnectLogin`, `CreateSession`) or a
 per-command URL override → **Web API**.
3. Otherwise → Web API **iff** `ServerInfo.versionIsAtLeast("8.1")` (raised to "8.2"
 for in-basket patient calls); below that → `CommandUnsupportedOperation` (no
 network call). Capability fallbacks: e.g. KeepAlive/Logout use legacy SOAP URNs
 exactly when `hasCapability("ECFLogout")` is false.

- Body = **flat JSON object** of the request record's properties (exact wire names in
 `message-spec.json`). No envelope; the URL path names the command.
- Binary properties → `multipart/mixed+json; boundary=<UUID>`: part 1 = the JSON
 (no Content-Disposition), then `Content-Disposition: inline; name="<prop>"` +
 `application/octet-stream` parts.
- Response: JSON object mapped onto the response record; multipart responses keyed by
 `content-disposition: name`.

### 2.2 SOAP facade (legacy URN commands)

```
POST <base>/httplistener.ashx
Content-Type: application/soap+xml; charset=utf-8
```

- Envelope: `soap:Envelope` with header elements `<Interconnect_ClientTime>`
 (RFC1123 GMT), optional `<Epic-Debug-ID>` in ns `urn:epicsystems-com:Interconnect.Headers`;
 optional `<ClientTimeZone ic:format="OLSON" ic:name="<tz>">`.
- **No SOAPAction / wsa:Action.** Body element = last dot-segment of the URN;
 its default xmlns = URN minus last segment; `Guid="<sessionToken>"` attribute when
 authenticated. Example:

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
 <soap:Header>
 <Interconnect_ClientTime xmlns="urn:epicsystems-com:Interconnect.Headers">Wed, 11 Nov 2026 19:00:00 GMT</Interconnect_ClientTime>
 </soap:Header>
 <soap:Body>
 <ClassSearch Guid="SESSIONTOKEN" xmlns="urn:epicsystems-com:EpicOnHand.2010-01.Services.InBasket">
...request properties...
 </ClassSearch>
 </soap:Body>
</soap:Envelope>
```

- Response parsing (client contract): assert `soap:Envelope`; skip `soap:Header`;
 the **first child of `soap:Body`** carries the response — its children are the
 response record's property elements (matched by name, unknown elements skipped;
 its own element name is **not** enforced, conventionally `<CommandName>Response` —
 a differently-named wrapper silently yields an empty response). Attributes map to
 properties only via `useAlternateKeyType` annotations (4 DTOs, none token-related).
- **No session-token echo**: `Guid="<sessionToken>"` is outbound-only (request
 operation element attribute); the token arrives inside `ECFLogin$Response.SessionInfo.Token`.
- Faults: `soap:Fault` →
 `faultcode`/`faultstring`/`detail>(applicationfault|internalfault @type)`.
- 150 cataloged URNs, `EpicOnHand.2007-07` … `2016` namespaces.

## 3. HTTP headers (all command requests)

| Header | Value | When |
|---|---|---|
| `Interconnect_SessionToken` | session token | every authenticated command |
| `Interconnect_ClientTime` | RFC1123 GMT now | always |
| `Epic-External-Request` | `true` | external-client flag (config) |
| `Authorization` | `<scheme> <token>` | OAuth2/OIDC commands (`EpicOAuth2Login`, `CreateSession`, …) |
| `Interconnect_WebBlobToken` | blob token | blob downloads |
| `X-Forwarded-For` | configured value | when configured |
| `Epic-Debug-ID` | correlation id | every command (both transports), when a client id is set: `<id>\|\|<appName>\|<platform>\|<deviceName>\|` if id ≥ 20 chars, else `'-'`-left-padded id variant. SOAP additionally embeds it as a SOAP header element (same ns as `Interconnect_ClientTime`) — i.e. SOAP sends it twice |

## 4. Serialization rules

**Final wire-name rule** (verified across all 15,963 catalog members):
- Default: getter-derived — `getEncryptedPassword` → `EncryptedPassword` (first letter
 case preserved, incl. multi-lowercase leads like `intEPTID`); `isXxx` → `IsXxx`.
- Override: `@transportName` annotation wins (244 members actually differ from the
 derived name; e.g. `getEncryptedPassword` → `Password`).
- Kotlin/field access: getter-less fields serialize under their raw field name
 (2,937 members, `fieldAccess: true`).
- 4 records have genuine intra-record wire-name collisions (e.g. `Login.Response`
 maps two getters to `ServerInfo`) — resolved at runtime by the real serializer's
 reflection order; a reimplementation should prefer the annotated/base-class one.
- 115 records are genuinely empty (`{}`) bodies (e.g. `CreateSession.Request`).
- Binary members (26 across 18 records, e.g. `UploadLog.LogData`) → multipart parts.

**Casing caveat:** the metadata rule keeps the first letter uppercase
(`EncryptedPassword`), but lowercase camelCase has been observed in one sample on the
wire. 🔴 Live observation against a real server must settle this.

- Dates: **seconds precision, no milliseconds, never epoch**. JSON writer:
 `yyyy-MM-dd'T'HH:mm:ss'Z'` (UTC, `SimpleDateFormat`, Locale.ROOT), written as a
 quoted JSON string — e.g. `"2026-11-11T18:00:00Z"`. SOAP/XML:
 `%04d-%02d-%02dT%02d:%02d:%02d` + `Z` or `±HH:mm` offset. Reader (both transports)
 is a lenient ISO matcher accepting `Z|[+-]HH:mm?` offsets, optional milliseconds,
 dashless forms; missing offset ⇒ client default TZ.
- Enums serialize via their string `.getValue`.
- Generics are erased in DEX — `List`/`Map` element types must be inferred from usage
 (77 void-typed members kept as-is in the catalog).
- Server errors: HTTP ≥400 → ASP.NET-style `{ExceptionType, Message,
 ExceptionMessage, StackTrace, InnerException}`; sentinel codes in reason phrase
 (`SESSION-EXPIRED`, `PATIENT-MERGED`, `PATIENT-WORKSPACE-CLOSED`,
 `INVALID-CLIENT-TIME`, `MOBILE-INTENTIONAL-THROW`) and `Message`
 (`INVALID-AUTH-TOKEN`, `INVALID-CLIENT-TIME`).
- `ServerMetrics` response header → performance instrumentation.

## 5. Authentication

1. `Epic.Mobile.Core.PreLogin` {} → `{PublicKey, …settings}`.
2. OAuth2 env: `EpicOAuth2Login`/`CreateSession` (Authorization header) → `{OAuth2Info: {loginToken, loginID}}`.
3. `Epic.Mobile.Core.Authentication.Login` wizard (`ECFLoginWizard$Request{CurrentStep,
 Workflow, …}`) — repeat per step (2FA, ChangePassword, job/department): carries
 `ECFLogin$Request{LoginID, EncryptedPassword/PasswordIV/EncryptedAESKey
 (RSA-OAEP vs PreLogin PublicKey), AuthToken+TokenType (ID|PIN|BiometricRefresh),
 Device/App/Locale/TimeZone/SystemInfo, OpenIDConnectCredential?}`.
4. `ECFLogin$Response` → `SessionInfo.Token` (+timeout) = Interconnect session token;
 biometric `RefreshToken`; `CheckPINToken`; `VinzInfo`; `UserInfo`; `ServerInfo`.
5. Session life: `KeepAlive` (capability-gated), `Logout`, `TerminateRemoteSession{applicationID, deviceID, key, token}`.

**Token chain:** OAuth2 loginToken → wizard → Interconnect SessionToken → biometric
refresh token (AndroidKeyStore RSA "HaikuFingerprintKeyPair") → optional client OIDC
(PKCE S256; redirects `com.epic.haiku.oidc://callback`, `https://mobile.epic.com/haiku/OIDC/redirect/`)
→ optional Nebula/Vinz JWT-bearer exchange.

**SSO variants:** server-side OIDC · client-side OIDC (browser CustomTab/MSAL) · SAML
(`epic://SAML/redirect` + `ValidateSAMLAuth`) · Hyperspace MFL QR/badge
(`GetMFLPreLaunchToken{codeChallenge}` → `GetMFLAuthCode{launchToken, codeVerifier}`).

**Re-auth triggers:** `SESSION-EXPIRED`, `INVALID-AUTH-TOKEN`, `INVALID-CLIENT-TIME`.

## 6. Realtime

- SignalR (stock wire). ENv1: `<base>/notificationservices/signalr`, hub `EventServiceHub`,
 methods `SubscribeTopic`/`Unsubscribe`/`Publish`, querystring `ClientID=<id>`.
 ENWS: `<base>/eventnotificationservice`, hub `EventNotificationHub`,
 `DynamicSubscribe`/`DynamicUnsubscribe`/`DynamicPublish`, WebSocket-only.
- Auth: `Authorization: Bearer` or hub `SetConnectionParameters {"session-token":
 {"token": <sessionToken>}}`. 300 s `KeepSessionAlive`. Ordered `EventCursor`;
 events: `{Type, Data, EventCursor, TimestampInUtc, ID="key-subkey"}`.
- Push (mobile): FCM w/ AES-256-GCM payloads; `RegisterDevice {deviceToken,
 environmentID, supportedDeliveryMethods}` → `{deviceTopic, encryptionKey, …}`;
 `DissociateDevice` on logout. Web/desktop clients use the SignalR channel directly.

## 7. Embedded Hyperspace WebView

- `Epic.Mobile.EHS.ObtainHSCredentials {CodeChallenge}` (PKCE S256) →
 `{LoginToken, PerimeterAuthenticationToken, HSUrl, Ticket}`.
- Load `HSUrl + "/AutoLogin.aspx"` with headers `X-Login-Token:
 "LoginToken|installationID|codeVerifier"`, `X-Initialization-Ticket: Ticket`.
- Perimeter token → `CookieManager.setCookie(hsWebUrl, token)` — the token string
 (from `ObtainHSCredentials$Response.PerimeterAuthenticationToken`) is set verbatim,
 i.e. it includes the server-chosen cookie name (`name=value; …`); the client does
 not construct a cookie name itself. 🔴 Exact cookie name only observable live.
- Per-screen: `Epic.Mobile.EHS.GetTicket {ViewParams, ActivityDescriptor/ID}`.
- Bridge: `WebMessageListener` JS object `__zCommunicator`, origin-restricted;
 JSON `Transmission{type: Request|Response|Event, name, id, data}`; services:
 ActivityMessage, CloseViews, ConnectionLost, DeactivateView, Heartbeat (5 s),
 OpenActivity, ViewActivated, ViewClosed.

## 8. Security posture

- TLS ≥1.2 enforced, cleartext refused. **No client certificate pinning** (no
 `CertificatePinner`, no embedded pins, platform trust store).
- Local store: SQLCipher; biometric token wrapper in `biometric_store` prefs
 (AndroidKeyStore-wrapped).
- WebViews: secure windows default, JS enabled, DOM storage on, file access off,
 remote debugging off; TLS errors fail closed.

## 9. Verification backlog (requires live observation)

🔴 Wire-name casing vs real server tolerance · actual `Unauthenticated` path-segment
handling · response element naming as emitted by real servers · date format emitted
by real servers for non-string fields · multipart boundary edge cases · server-side
client fingerprinting/tolerance for a third-party client.

---

*All statements in this specification were derived from the documented client version
alone; no Epic or hospital servers were contacted in producing it.*
