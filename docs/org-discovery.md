# Epic Haiku 11.9.2 (Android) — Server / Organization Discovery and Connection Configuration

Class and field names are as they appear in the shipped client (short single-letter
package names such as `e10`, `vb0` are the client's own internal names; `L...;` forms
are VM type descriptors, reproduced verbatim).
(note: those files contain the bytecode toolkit DEBUG spam; the useful part is the `=== STRING:` / `ref:` blocks at the end).

---

## 1. Executive answer: there is no "find your organization" directory lookup

Haiku does **not** search a directory of hospitals by name. There is no org-search HTTP call in the
app (verified: no client strings exist for "find your organization", org search parameters,
server-name search, etc.; the only `organization*` strings belong to the MSAL/Microsoft broker
library or resource names for "Change Organization" = switching between *already-configured*
environments). Instead the hospital → device binding is delivered by an **encrypted configuration
document ("econfig")** that the user obtains out-of-band:

1. **MDM managed configuration** — Android `RestrictionsManager` application restrictions key
 `EpicMobileConfigurationURL` (+ `EpicApiKey`, `EpicMDMName`, `EpicMobileDeviceID`,
 `EpicNotificationsManaged`, `EpicShowLoginBanner`, `EpicShowEula`). See `Lrk/e1;`, MDM
 testing utility screen listing these keys) and `Lcom/epic/haiku/android/authentication/p1;->tc(boolean)`
, the login activity reads restrictions on `APPLICATION_RESTRICTIONS_CHANGED`).
 Resource string: *"This application is missing the key 'EpicMobileConfigurationURL'."*, and
 *"The scrambled URL holding the location of the econfig file on the Interconnect server"* —
 i.e. the econfig itself is hosted **on the hospital's own Interconnect web server**.
2. **Deep link**: `epic://handheld/config/<scrambled-url>` (Canto uses scheme `cipe://`). Parsed in
 `Lcom/epic/haiku/android/configuration/k2$a;->a(Context, Uri)`:
 - scheme must start with `epic` or `cipe`
 - host must be `handheld`
 - path must be exactly 2 segments, first segment literally `config`
 - second segment is **URL-decoded** (`com.epic.util.h.g.n(...)`) into a `java.net.URL` — the econfig URL.
 Sent to users by the hospital IT department (typically via email/QR). MDM `EpicMobileConfigurationURL`
 actually contains one of these full `epic://handheld/config/...` links (`p1.nb(String, boolean)`
 re-parses it with `k2.kC(...)`).
3. **Manual entry**: Econfig Testing utility (`com.epic.clinical.controls.testingutility.EconfigTestingUtilityFeature`,
 `res/layout/econfig_testing_utility.xml`, resource strings "Econfig URL", "Custom Configuration",
 "Configuration entered in the app settings"). `p1.wc(String)` also shows hand-typed URL handling
 (`http://` / `https://` prefix checks).
4. On every app start stored configs can be refreshed (`refresh_all_econfigs` in the `k2` fragment args;
 string constants `ECONFIG_REFRESH_ALL`, `ECONFIG_URL_INSTANCE_KEY`, `ECONFIG_IS_FROM_MDM`).

`mobile.epic.com` is **not** a discovery API — it is Epic's *reverse-proxy for login redirects*
(see §5).

---

## 2. Downloading the econfig document

Classes: `Lcom/epic/haiku/android/configuration/k2;` (no-UI fragment `urlHandlerNoUI`),
`...configuration.l2;` (viewmodel), `...configuration.s0;` (downloader, extends `Lt60.b;`),
`Lt60.b$b;` (raw HTTP), `...configuration.v1;` (ConfigurationManager), `Le10.b;` (`Le10.c;` containers).

### 2.1 HTTP request

`Lt60.b$b;->beginWork` is a plain **HTTP GET** to the config URL taken from the deep link:

```
GET <econfig-url> HTTP/1.1
If-Modified-Since: <previously-stored 'Generated' timestamp, if known>
If-None-Match: "<previously-stored ETag/version, if known>"
```

- Requires HTTPS (`com.epic.util.y.a` rewrite when a "test" mode flag is on; `SSLException("Https is required")`
 otherwise in prod mode).
- Read timeout 30 s, no auth, no body.
- HTTP **304** is treated as "already up to date" (`t60.b$c` exception; logged as
 *"Downloaded configuration from %s but the existing file is already up to date (timestamp: %s)"* in
 `l2.C0(...)`).

### 2.2 Response — the outer (encrypted) document

Response body is parsed by `s0.j(byte[] body, Map headers)` →
`new e10.c(e10.b.f(XmlPullParser over body), body, sourceURL, firstETagHeader)`.

`Le10.b;->f(XmlPullParser)` reads an XML document with these elements:

```xml
<Guid> <!-- configuration GUID (required) -->
<Name> <!-- human-readable config name (required) -->
<Generated> <!-- timestamp string -->
<Iterations> <!-- int -->
<Salt> <IV> <!-- base64 (hex) -->
<PasswordHash> <!-- hash of the config password, used to verify it -->
<Data> <!-- encrypted inner configuration XML -->
```

Failure messages in `Le10.b;->f`: `"configuration is missing guid"`, `"configuration is missing data"`,
`"configuration is missing name"`.

**Decryption** (`Le10.b.h(b, password)`):
- KDF `PBKDF2WithHmacSHA1`, 256-bit key, iteration count from `<Iterations>`, salt from `<Salt>`,
 password = the *configuration password* the hospital IT gave the user (resource string:
 *"A password is required to add configuration %1$s."*). The viewmodel `l2` suspends in a
 password-prompt state (`l2$d$f`; `D0(String)` resumes with the typed password).
- Legacy fallback: `"Epic Scramble"` static salt/key (`com.epic.util.h` scramble) via
 `Le10.b.i(b, password)` when `<PasswordHash>` is empty; used when reading old `*.econfig`
 files (`v1.X(File)` re-writes the file decrypted after successful legacy decrypt).
- Cipher: `com.epic.util.io.a` (AES wrapper) over `Data`, IV from `<IV>`.

Inner `Data` XML: `Le10.b.j(byte[])` → `Le10.a` = **Configuration** node.

### 2.3 The Configuration document model (package `e10`)

| Class | Role |
|---|---|
| `Le10.f;` | abstract node: fields `Guid`, `Name`, `Description`, `Settings` (string→string map), children `Group`/`Environment`. Parser switch on exactly these element names (`k(XmlPullParser)`). |
| `Le10.a; extends f` | **Configuration** root (`f$a.Configuration`); adds `<Environment>` / `<Group>` children and setting `RequirePasswordForEdit`. |
| `Le10.j; extends f` | **Group** node (`f$a.Group`). |
| `Le10.g; extends f` | **Environment** node (`f$a.Environment`) — one hospital login target (details §4). |
| `Le10.b;` | encrypted container (above). |
| `Le10.c;` | download result: (b container, raw bytes, source `java.net.URL`, ETag string). |
| `Le10.h;` | app interface: `d` current environment, `w0` environments, `A1(String)` find environment by GUID/name, `X1(String)`, `n2(String, e10.i)` selection callback. |
| `Le10.i;` | enum `EnvironmentSelectionType`: `NONE, LOGIN_SCREEN, LOGIN_SCREEN_QUICK_BUTTON, CHANGE_CONTEXT_MENU, QUICK_BUTTON_CHANGE_CONTEXT_MENU, NOTIFICATION, EXTERNAL_LINK`. |
| `Le10.k;` | `Exception` subclass for config errors. |

So a single econfig file may declare **multiple environments** (multiple connection entries, e.g.
PROD/TEST) grouped in `<Group>`s.

### 2.4 Where the selected values are stored

`v1` (ConfigurationManager, `com.epic.haiku.android.configuration`):
- every downloaded config saved as `<filesDir>/*.econfig`;
- index JSON in default `SharedPreferences` key `"configuration"` (collection class `v0` of
 records `u0` with GUID, timestamp, ETag, flags);
- selected config/environment: `"configuration_guid"`, `"environment_guid"`,
 `"last_environment_selection_type"`; recent environments list `"recent_environments_key"`.
- GUID for a typed-in custom config is derived `UUID.nameUUIDFromBytes(hash(url))` (`v1.G`).

---

## 3. The Environment/connection model (`Le10.g;`)

Each `<Environment>` node (parsed in `Le10.g.j(parser)` / `k(parser)`):

```xml
<Environment>
 <Name>...</Name><Guid>...</Guid><Description>...</Description>
 <Address>https://hospital.example.com/Interconnect/</Address>
 <CustomSettings>
 <Key/><Value/>... repeated
 </CustomSettings>
</Environment>
```

`Address` = the hospital **Interconnect** base URL (user-visible string confirms:
*"The URL entered does not appear to be pointed to a functioning Interconnect instance"*.)

### 3.1 Auth-mode enum (`Le10.g$a;`)

`fromValue(int)` of the `LOGIN.AUTH_TYPE` custom setting:

| value | name | meaning |
|---|---|---|
| 0 | `DEFAULT` | username/password against Interconnect (classic form login) |
| 1 | `OAUTH2` | Epic-hosted OAuth2 (web form / SAML-ish flow) |
| 2 | `OPENID_CONNECT` | OpenID Connect via hospital IdP (MSAL support for Azure AD via `InitiateOpenIDConnectLogin$MicrosoftAuth`) |

Unparseable ⇒ `DEFAULT`.

### 3.2 Ternary flag enum (`Le10.g$b;` implements `com.epic.util.z` value provider)

`No=0, Yes=1, UseAppDefault=2` — used for `HEADER.DISASSOCIATE_ON_LOGOUT`.

### 3.3 Observed `CustomSettings` keys (static accessors in `Le10.g;`)

| key | accessor | default |
|---|---|---|
| `LOGIN.AUTH_TYPE` | `q` | `DEFAULT` |
| `LOGIN.REMEMBER_LAST_USER` | `D(...)` | false |
| `LOGIN.DISABLE_REVEAL_PASSWORD` | `o(...)` | false |
| `HEADER.DISASSOCIATE_ON_LOGOUT` | `p` | `UseAppDefault` |
| `OAUTH2.MAINTAIN_SESSION` | `w` | false — if true, Authorization header is attached to more commands (see: `b`: `if ((!q.w) && (!t60.i.A.contains(command)))` drop the `Authorization` header) |
| `OIDC.ENABLE_SSO` | `x` | false |
| `OAUTH2.CLIENT_ID` | `z` | must be `OAUTH2` auth-type AND client id one of the two hard-coded values `80FFB3235CCD4B41891617711D958594` or `8B3A7DA58CF64F699BE96F9306F7A3FD` (Epic-published client IDs; z returns true when matched) |
| `BARCODING.ALLOW_CAMERA_SCANNING` | `n(...)` | 1 |
| `BARCODING.OVERRIDE_CAMERA_SCANNING_VENDOR` | `B(...)` | 0 |
| `BARCODING.OVERRIDE_CAMERA_SCANNING_LICENSE_KEY` | `A(...)` | "" |
| `DESKTOP.HYPERSPACE_URL` | `y(...)` | "" (launch Hyperspace-web on desktop) |

OAuth2 settings keys exist app-wide: `OAUTH2.AUTHORIZATION_ENDPOINT`, `OAUTH2.TOKEN_ENDPOINT`,
`OAUTH2.CLIENT_ID`, `OAUTH2.CLIENT_SECRET`, `OAUTH2.LOGIN_BUTTON_TITLE`;
data classes carry `AuthorizationURL`, `ClientID`, `ClientSecret`, `RedirectURIFormat`,
`EventNotificationClientID` fields (the OIDC/OAuth2 `Settings` responses).

### 3.4 Base-URL normalization (`Le10.g.r`)

`r` makes the "service" URL used by all webservice traffic:
- if `Address` ends with `/httplistener.ashx` (17 chars) it is stripped;
- otherwise a trailing `/` is appended if missing;
- parsed to `java.net.URL` (returns null on malformed);
- `vb0.d.a.o` (test/SSL-bypass flag) rewrites via `com.epic.util.y.a`.

### 3.5 Server-side config: `com.epic.communication.ServerInfo`

`Lcom/epic/communication/ServerInfo;`; used from `Connection.getServerInfo`):
ctor *(encryptionKey, version, isAndroidSupported, capabilities Set, readOnly, loginMessage,
isUserLoggingEnabled, userHasEmbeddedHSAccess, envID)*. It comes from the login response —
command names `Epic.Mobile.Core.Authentication.ServerInfo`, `...ClientConfiguration`,
`...SystemInfo`, `...ApplicationInfo` exist. Capability model:

- `hasCapability(Capability)` checks membership in the set **or** `versionIsAtLeast(capability.getVersion)`.
- Server version constants: `7.8` (2010) … `8.3` Gasherbrum, `8.4` 2018, `8.7/8.8/8.9` 2019,
 `9.4` 2020-Aug, `10.2.0` 2022-May, `10.6.0` 2023-Aug.
- Capability enum `Lcom/epic/haiku/communication/model/HaikuCapability;` (~400 values,
 e.g. `MOBILE_ECF_LOGOUT_CAPABILITY`, `BADGE_MANAGER_WEBSERVICE`, `HAIKU_PIN_LOGIN`);
 each value carries a `getValue` capability string + `getVersion` minimum server version.

---

## 4. What `mobile.epic.com` actually is (and is not)

All `mobile.epic.com` references:

| refs | purpose |
|---|---|
| `Lgd/a;` | **SAML AssertionConsumerService/redirect URL**: `https://mobile.epic.com/<appname-lowercase>/SAML/redirect` and `https://mobile-test.epic.com/<appname>/SAML/redirect`; app name from `vb0.d.a.f` = `haiku` / `rover`. `gd.a$a` builds a full SAML 2.0 `AuthnRequest` XML (with `epic:` extension `Attribute`s OS/OSVersion/ApplicationName/ApplicationVersion/DeviceModel/DeviceManufacturer/InstallIdentifier) sent as `?SAMLRequest=<urlencoded, deflated>` to `SAMLAuthSettings.getWebFormBaseURL` (or `getExternalAppBaseURL` for the external-app workflow) — `epic://SAML/redirect` is the app-scheme variant of the same redirect. `gd.a$c$a` matches the redirect-back and extracts the `SAMLResponse` query parameter (limit 256 KiB). |
| `Lcom/epic/oauth2/m$a;` | **OIDC redirect URIs** enum, 2 values: `NAMESPACED_NATIVE_APP_CUSTOM_SCHEME` = `com.epic.haiku.oidc://callback` (or `com.epic.rover.oidc://callback`), and `HTTPS_UNIVERSAL_LINK` = `https://mobile.epic.com/haiku/OIDC/redirect/` (test: `https://mobile-test.epic.com/haiku/OIDC/redirect/`). `com.epic.oauth2.m.b(token)` decodes a failed ID-token for logging. |
| `Lcom/epic/haiku/android/externalLinks/ExternalLinksActivity;` | marketing/support links UI. |
| `Ll40/d;` / `Ll40/d$a;` | companion to `SuccessfulAuthAssertionProperties(id, issueInstant, issuer, userAtDomain, recipient, audience, authnInstant, authnContext, transactionId)` — the SAML assertion properties after web-form auth. |
| `Lb20/d0;->P` | dev/test-only Elsa/HyperSpace-web file service: `https://hsw-fs-test.epic.com/ElsaSDB`, `https://prj271932.epic.com` + `.../AutoLogin.aspx`. Internal Epic test scaffolding, **not** part of the customer flow. |
| `Lp60/d;` / `Lp60/j;` | permission-test WebView: loads bundled HTML with base URL `https://permission-test.epic.com/` and a `PermTestBridge` JS interface — an in-app device-permission self-test page, no discovery. |

**Conclusion**: `mobile.epic.com` hosts Epic-side *login redirect receivers* (OIDC universal-link
callback, SAML redirect landing page) that hand the flow back to the app. It is shared across all
hospitals and knows nothing about per-hospital servers (the hospital identity travels inside the
assertion/issuer/`Destination` of the SAML request and the OIDC settings from the hospital). The
only URLs that "discover" a hospital are the `epic://handheld/config/...` link contents (→ the
hospital's own Interconnect-hosted econfig).

---

## 5. Wire protocol against the hospital server

### 5.1 Legacy ECF (XML) transport — `httplistener.ashx`

`Lt60.a;` (abstract client,; for subclass `Lt60.i;`):

```java
base = environment.r; // §3.4
this.n = new URL(base, "httplistener.ashx"); // t60.a.s(URL)
```

`Lt60.i` (pre-login processor) is constructed as `(e10.g environment, String clientSecret,
String sessionId)` and its command allow-lists are:

```java
Z = { "Epic.Mobile.Core.PreLogin", "Epic.Mobile.Core.Authentication.Login",
 "Epic.Mobile.Core.TerminateRemoteSession", "Epic.Mobile.Core.SearchRecordsPreLogin" }
A = { "Epic.Mobile.Core.Authentication.EpicOAuth2Login", "Epic.Mobile.Core.CreateSession" }
```

Path overrides (`Lt60.i.d(...)`):
- `Epic.Mobile.Core.Authentication.EpicOAuth2Login` → `internal/Auth/OAuth2/EPICOAUTH2LOGIN`
- `Epic.Mobile.Core.CreateSession` → `api/epic/2017/Auth/OAuth2/CREATESESSION`

HTTP request (`CommandWebServiceOperation.connectionFromCommand`):

```
POST <Address>/httplistener.ashx
Accept: <response content type>
Content-Type: <request content type>
Interconnect_ClientTime: <formatted UTC now>
Interconnect_SessionToken: <token, only when command requires authentication>
Epic-External-Request: true (when external workflow)
X-Forwarded-For: <client IP chain>
Authorization: <"Bearer …"/other token string>, only if OAUTH2.MAINTAIN_SESSION
 or command ∈ {EpicOAuth2Login, CreateSession} (t60.i.b)
```

XML envelope is written by `t60.a.f(command)` with version **"8.1"**, `"Mobile"`/`"Internal"`
application/product identifiers in the root, namespace derived from the command name suffix
(`t60.a.u(name)` splits off the last dot-segment), `"Unauthenticated"` marker when
`!getRequiresAuthentication`.

Capability fallback (in `t60.a.c/e/m`): when the server lacks
`MOBILE_ECF_LOGOUT_CAPABILITY`, KeepAlive/Logout degrade to legacy SOAP URNs
`urn:epicsystems-com:EpicOnHand.2010.Services.Login.KeepAlive` and
`urn:epicsystems-com:EpicOnHand.2009-08.Services.Login.Logout` (`q60.d` legacy command class);
commands whose minimum server version is unmet raise `CommandUnsupportedOperation`.

### 5.2 Command Web API (JSON) transport

`CommandWebAPIServiceOperation` — selected by `t60.a.m`
when the command carries a web-API name, or is in the pre-login/authallow list:

- URL: `<Address>/internal/<Mobile>/<Internal>/<CommandName>` —
 `String.format("internal/%s/%s/%s", product, area, commandName)`, e.g.
 `internal/Mobile/Internal/Epic.Mobile.Core.Authentication.Login`
 (confirmed by `BlobDownloadRequest` = `internal/Mobile/Internal/Epic.HandWritten.Mobile.DownloadWebBlobFile`).
- Request: `Content-Type: application/json; charset=utf-8`; optional multipart
 `multipart/mixed+json; boundary=<uuid>` where part 1 is the JSON request and following parts are
 octet-stream blobs (`content-disposition: name="..."`).
- Response: `application/json`, or `multipart/mixed+json` (first part JSON, rest blobs).
- Error reason phrases mapped to client errors: `SESSION-EXPIRED`, `PATIENT-MERGED`,
 `PATIENT-WORKSPACE-CLOSED`, `MOBILE-INTENTIONAL-THROW`, `INVALID-CLIENT-TIME`.
- Server metrics returned in header `ServerMetrics` → recorded.

### 5.3 Authentication commands (Epic.Mobile.Core.Authentication.*)

Command names under `Epic.Mobile.Core.Authentication.*`:
`Login`, `PreLogin` (core), `ValidateUser`, `ValidateOpenIDConnectAuth`,
`ValidateOpenIDConnectIDToken`, `ValidateMicrosoftAuth`, `ValidateSAMLAuth`,
`ValidateWebFormAuth`, `GetOpenIDConnectSettings` (V1/V2 responses; request carries
*(String, String, String, boolean)* e.g. user/loginID/…; `MicrosoftAuth` object for Azure AD),
`GetSAMLAuthSettings` (response `SAMLAuthSettings(webFormBaseURL=…, externalAppBaseURL=…,
audience=…, domain=…, partnerName=…, tenantID=…, organizationDisplayName=…, workflow=…)`,
cf. `gd.a$b$b.b`), `GetWebFormAuthSettings`, `GetNative2FASettings`, `GetPINValidationSettings`,
`EpicOAuth2Login`, `ResolveMultipleContexts` (multi-department/job selection), `CheckPINReauth`,
`CheckBarcodeReauth`, `StoreBiometricValidationResult`, `ApplicationInfo`, `ClientConfiguration`,
`ServerInfo`, `SessionInfo`, `SystemInfo`, `UserInfo`, `AuthToken`, `EpicNative2FA`, `OTP*`.

So the auth-method negotiation happens per-server: the app calls the relevant `Get*Settings`
command over the transports above and branches into OIDC (with the two redirect URIs from §4),
SAML web-form (`gd.a$b` AuthnRequest → IdP URL → `SAMLResponse` back through
`epic://SAML/redirect` or `mobile[.test].epic.com`), or classic user/password into `Login`.

---

## 6. End-to-end flow: "I work at hospital X" → session at hospital X

1. **Configuration delivery (no Epic server necessarily involved yet)** — user taps an
 `epic://handheld/config/<url-encoded-hospital-econfig-url>` link (hospital IT / Epic provisioning
 email) **or** MDM pushes `EpicMobileConfigurationURL` containing such a link.
 - `URLHandlerActivity` (`Lcom/epic/haiku/android/configuration/`) → confirm dialog showing the
 host (*"A configuration file will be downloaded from %1$s. Only add it if you recognize this
 source."*).
2. `k2` fragment → `l2.z0(url, isFromMDM=…, refreshAll=…)` → **CALL 1**:
 `GET <econfig-url>` with `If-Modified-Since`/`If-None-Match` (§2.1). Response = encrypted econfig
 XML (§2.2). If a password is required the app prompts (*"A password is required to add
 configuration %1$s."*) and decrypts with `PBKDF2WithHmacSHA1` (§2.2).
3. The decrypted `<Configuration>` (§2.3) enumerates `<Environment>` nodes (usually one per
 hospital connection). GUIDs + selection are persisted (§2.4). The login screen shows the
 environment name(s) ("Change Organization" switches between stored environments; no directory
 search exists).
4. User selects environment X → `e10.h.d` returns X; derived EpicSettings
 (`com.epic.communication.EpicSettings`, has `getLoginID`, capabilities, etc.) are bound to
 `com.epic.communication.Connection` with base URL `X.r` = `Address` ± `/httplistener.ashx`
 (§3.4).
5. ** CALL 2 (pre-login)**: `POST <Address>/httplistener.ashx` (or
 `/internal/Mobile/Internal/...`) with `Epic.Mobile.Core.PreLogin` in the XML/JSON envelope of
 §5.1/§5.2 — allowed pre-auth, sends client info
 (`gd.a$a`-style device attributes: OS, OSVersion, ApplicationName, ApplicationVersion,
 DeviceModel, DeviceManufacturer, InstallIdentifier).
6. **Auth-method negotiation, CALLS 3..n (one of)**:
 - `OPENID_CONNECT`: `Epic.Mobile.Core.Authentication.GetOpenIDConnectSettings` → authorization
 URL/`RedirectURIFormat` → browser (or MSAL for Azure AD) → redirect lands on
 `com.epic.haiku.oidc://callback` or `https://mobile.epic.com/haiku/OIDC/redirect/` →
 `Epic.Mobile.Core.Authentication.ValidateOpenIDConnectAuth` / `...ValidateOpenIDConnectIDToken`;
 - `OAUTH2`/SAML web-form: `GetSAMLAuthSettings` → `AuthnRequest` to `webFormBaseURL?SAMLRequest=…`
 (§4) → `SAMLResponse` returned to `epic://SAML/redirect` / `mobile.epic.com/haiku/SAML/redirect`
 → `ValidateSAMLAuth` (assertion properties in `l40.d`);
 - `DEFAULT`: username/password through `Epic.Mobile.Core.Authentication.Login`.
7. **CALL last in handshake**: `Login`/`CreateSession` response carries
 `com.epic.communication.ServerInfo` (§3.5): server version range, `isAndroidSupported`,
 capability set, `readOnly`, `loginMessage`, `encryptionKey`, `envID`, `userHasEmbeddedHSAccess`.
 From here on every command is capability- and version-gated
 (`ServerInfo.versionIsAtLeast`, `HaikuCapability`).
8. Session commands: `POST <Address>/httplistener.ashx` with header `Interconnect_SessionToken`
 (or `internal/Mobile/Internal/…` JSON); keep-alive `Epic.Mobile.Core.KeepAlive` (or legacy
 `urn:epicsystems-com:EpicOnHand.2010.Services.Login.KeepAlive`); logout
 `Epic.Mobile.Core.Logout` / legacy `...EpicOnHand.2009-08.Services.Login.Logout`; blob/file
 download via `Epic.HandWritten.Mobile.DownloadWebBlobFile` with `Interconnect_WebBlobToken`.

---

## 7. Uncertainties / not fully verified

- The exact inner `<Configuration>` XML produced by hospital provisioning (element order, exact
 required settings) is inferred from the parser switch in `Le10.f.k`; the settings key universe
 is larger than the 11 keys read in `Le10.g` (other code paths read other keys; e.g.
 `hsWebUrlFromEconfig`, `ECONFIG_*` prefs).
- Whether `Epic.Mobile.Core.PreLogin`'s response includes a `ServerInfo` before authentication was
 not traced end-to-end; the capability-gated fallback shows the login/logout pair has both ECF
 and legacy variants at `Login`-workflow scope (t60.i `k` only accepts
 `LoginWorkflowDefault/Start` steps).
- `com.epic.util.y.a(url)` rewriting (test-mode URL mangling) and `vb0.d` global state (which
 contains app id, app name, "o" SSL-test flag, "e" stored econfig URL) were not traced.
- MDM `EpicMobileConfigurationURL` value format: evidence shows it is parsed by the same
 `epic://handheld/config/...` parser (`p1.nb` → `k2.kC`), but a hospital may alternatively host the
 raw `.econfig` URL if their MDM stores a full link; the resource string
 *"Redirect URI for configuration json file"* hints at additional OIDC-broker config not explored.
- The `pbk` "Epic Scramble" key derivation (`com.epic.util.h`) internals were not traced.
- No references to `.well-known/openid-configuration` exist in Epic code (`well-known`
 strings live only in the MSAL/Azure-AD library); Epic OIDC metadata comes from the
 `GetOpenIDConnectSettings` command instead. Confidence: high.
- `Invalid endpoint, must start with http:// or https://: ` (task premise) belongs to
 `io/opentelemetry/exporter/internal/ExporterBuilderUtil;` — an OTel exporter check, **not** Haiku's
 URL validation. The Epic-side analogue strings are
 `"Invalid endpoint, must be a URL: "` and `"Invalid endpoint: "` (refs not pinned — likely in the econfig/config-url validators).

## 8. Reimplementation pointers

- Discovery = "bring your own URL": implement the `epic://handheld/config/<url>` link format and the
 econfig GET (304-aware, If-None-Match), and the two container formats:
 PBKDF2/AES container (`Iterations/Salt/IV/PasswordHash/Data`), optional legacy "Epic Scramble".
- Model = `Configuration → Group* / Environment*` with `Address` + flat key/value settings;
 serialize/parse the XML in §2.3 with the key names of §3.3.
- Transport = POST `<Address>/httplistener.ashx` (XML envelope, version 8.1) or
 `<Address>/internal/Mobile/Internal/<CommandName>` (JSON), headers per §5.1; keep
 `Interconnect_ClientTime` and `Interconnect_SessionToken` semantics (server maps error
 `INVALID-CLIENT-TIME`, `SESSION-EXPIRED`).
- Redirect endpoints to replicate for OIDC/SAML tests: `https://mobile.epic.com/haiku/OIDC/redirect/`,
 `.../haiku/SAML/redirect`, custom scheme `com.epic.haiku.oidc://callback`, `epic://SAML/redirect`
 (test twins on `mobile-test.epic.com`).

---

## Resolved uncertainties (verification pass 2)

### 1. econfig inner XML schema (complete, from `Le10/f;` switch —

Parser stack: `le0.j` (XmlPullParser wrapper) → `le0.f` (typed reader: `O0`=nextTag-onto-element,
`h`=current element name, `D0`=element text as String, `n0`=element text Base64-decoded via
`com.epic.util.d.a` = `le0.b.b`, `y0`=int, `q0`=boolean, `z`=is-empty-element, `a0`=skip subtree,
`F0`=consume EndElement). **Everything is element-based — attributes are never used.** Root element
name is not validated; the parser walks by element names. Unknown elements are skipped (`f.j` calls
`a0`), except inside `Environment` where `j` explicitly skips unknown children.

Schema sketch (annotations inline):

```xml
<Configuration> <!-- e10.a (Le10/a;) i=f$a.Configuration -->
 <Guid>...</Guid> <!-- e10.f.f -> configuration GUID -->
 <Name>...</Name> <!-- e10.f.getName -->
 <Description>...</Description> <!-- e10.f.d -->
 <Settings> <!-- e10.f.h: HashMap<String,String>; children = <Key>text</Key> pairs -->
 <RequirePasswordForEdit>true|false</RequirePasswordForEdit> <!-- e10.a.o reads boolean -->
 <any-key>value</any-key>
 </Settings>
 <Group> <!-- e10.j (Le10/j;) i=Group; recursive, same element set as Configuration -->
... (Guid/Name/Description/Settings/Group/Environment)...
 </Group>
 <Environment> <!-- e10.g (Le10/g;) i=Environment; NO nested groups (g.g throws) -->
 <Guid>...</Guid> <Name>...</Name> <Description>...</Description>
 <Address>...</Address> <!-- e10.g.getAddress; g.r normalizes to base http(s) URL,
 strips trailing "/httplistener.ashx", appends "/" if needed -->
 <Settings> <LOGIN.AUTH_TYPE>0|1|2</...> <!-- g.q -> g$a: 1=OAUTH2, 2=OPENID_CONNECT, else DEFAULT -->
 <OAUTH2.CLIENT_ID>GUID</...> <!-- z true when AUTH_TYPE==OAUTH2 and CLIENT_ID in
 {80FFB3235CCD4B41891617711D958594,
 8B3A7DA58CF64F699BE96F9306F7A3FD} (Epic-shipped clients) -->
 <OAUTH2.MAINTAIN_SESSION>bool</...> <!-- g.w -->
 <OIDC.ENABLE_SSO>bool</...> <!-- g.x -->
 <LOGIN.DisableRevealPassword-variant> <!-- g.o: literal key "LOGIN.DISABLE_REVEAL_PASSWORD" -->
 <LOGIN.REMEMBER_LAST_USER>bool</...> <!-- g.D -->
 <HEADER.DISASSOCIATE_ON_LOGOUT>int</...> <!-- g.p -> g$b 0=No,1=Yes,2=UseAppDefault -->
 <DESKTOP.HYPERSPACE_URL>url</...> <!-- g.y -->
 <BARCODING.ALLOW_CAMERA_SCANNING>int</...> <!-- g.n, default 1 -->
 <BARCODING.OVERRIDE_CAMERA_SCANNING_VENDOR>int</...> <!-- g.B, default 0 -->
 <BARCODING.OVERRIDE_CAMERA_SCANNING_LICENSE_KEY>str</...> <!-- g.A -->
 <any-key>value</any-key>
 </Settings>
 <CustomSettings> <!-- e10.g.j override -> C; e10.g.v: HashMap (separate from Settings) -->
 <Key>name</Key> <!-- read directly (no wrapper element); skipped if empty -->
 <Value>...</Value> <!-- optional, defaults "" when absent -->
 (Key/Value pairs repeat; unknown children skipped)
 </CustomSettings>
 </Environment>
</Configuration>
```

Environment selection persistence (`v1`): prefs `configuration_guid`, `environment_guid`,
`last_environment_selection_type` (`e10.i`: NONE=0, LOGIN_SCREEN=1, LOGIN_SCREEN_QUICK_BUTTON=2,
CHANGE_CONTEXT_MENU=3, QUICK_BUTTON_CHANGE_CONTEXT_MENU=4, NOTIFICATION=5, EXTERNAL_LINK=6).

Still ambiguous: the settings key universe read by other consumers (keys are free-form `HashMap`; only
the 11 keys above are statically pinned from `e10.g`, plus `RequirePasswordForEdit` from `e10.a`);
`CustomSettings` semantics (consumers of `g.v` beyond the readers above) unresolved.

### 2. econfig container + download + legacy Scramble (full parameters)

Container XML — `e10.b` (Le10/b;, parsed by `e10.b.f(le0.f)`, fed from raw HTTP body):

```xml
<...container> <!-- root name not validated -->
 <Guid>...</Guid> <!-- required else e10.k "configuration is missing guid" -->
 <Name>...</Name> <!-- required else "configuration is missing name" -->
 <Generated>timestamp</Generated> <!-- e10.b.b: parsed with com.epic.util.n for freshness compare -->
 <Iterations>int</Iterations> <!-- PBKDF2 iteration count -->
 <Salt>base64-string</Salt> <!-- ALSO the fallback *password* when none supplied (see below) -->
 <IV>base64</IV>
 <PasswordHash>base64</PasswordHash><!-- presence selects legacy Scramble path -->
 <Data>base64</Data> <!-- the inner Configuration XML, Base64 (com.epic.util.d custom alphabet
 "A-Za-z0-9+/=", le0.f.n0 -> le0.b.b), required else "missing data" -->
</...container>
```

Decryption `e10.b.h(container, password)` — two modes:

1. **PBKDF2 mode** (when `PasswordHash` is absent/empty):
 - password = caller-supplied; **if null/empty, the raw `<Salt>` element text is used as the password**.
 - key: `SecretKeyFactory("PBKDF2WithHmacSHA1")`, `PBEKeySpec(password, Base64Decode(Salt),
 Iterations, keyBits=256)` → AES-128...256-bit key (256 bits as passed).
 - cipher: `com.epic.util.io.a` = CipherInputStream built by `com.epic.util.a.g(2, key, iv)` →
 **AES/CBC/PKCS5Padding**, IV = Base64Decode(`<IV>` element). Decrypt → inner Configuration XML
 (`e10.b.j` → `e10.a`).
2. **Legacy "Epic Scramble" mode** (`e10.b.i`, when `PasswordHash` present):
 - verify: `e10.b.e` compares stored `PasswordHash` with `com.epic.util.h.i(password,"Epic Scramble")`
 i.e. Base64(first16byteofUTF8hex(SHA-1(UTF-16LE(password + "Epic Scramble"))))
 (`h.j`: `com.epic.util.e.d(...)` uppercase hex string, `.getBytes("UTF-8")`, arraycopy 0..15).
 Default constant `e10.b.l` = same hash of empty password.
 - AES key = `h.j(password, "Epic Scrambler")` (note: salt string has a trailing `r`) → SHA-1 of
 UTF-16LE(password+"Epic Scrambler"), hex-string ASCII bytes, first 16 bytes → AES-128 key.
 - cipher: AES/CBC/PKCS5Padding via `com.epic.util.a.g(2,key,iv)`; **static IV = h.g's IV =
 {0xD2,0xAD,0xBE,0xEF, 0xD2,0xAD,0xBE,0xEF,...} (0xD2ADBEEF repeated 4x, 16 bytes total)**
 (`e10.b.k` = `com.epic.util.h.l` = instance `h.g`; also carries unused key literal bytes ASCII
 "9FE3C86E96C5CC9C" and static field `h.e` = "Epic Scrambler\0\0").

Download (`com.epic.haiku.android.configuration.s0 extends t60.b`; runner `t60.b$b` SyntheticCommand):
plain `HttpURLConnection`, 30 s read timeout, conditional GET via `setIfModifiedSince(last-modified
Date)` + header `If-None-Match: <etag>`; on 304 → `t60.b$c` "unchanged" error; ETag is recovered from
response headers (filter `equalsIgnoreCase("ETag")`) and persisted. Test-mode URL mangling applies when
`vb0.d.a.o` (SSL-test flag; refuses plain HTTP, wraps socket via
`webservices/execution.j` SSLException hook; same test rewrite `com.epic.util.y.a` used in `e10.g.r`).

URL provisioning → `com.epic.haiku.android.configuration.k2.kC(context, uri)` (companion `k2$a.a`):
scheme must start with `epic` or `cipe`; host must be `handheld`; path must be exactly 2 segments
`/handheld/config/<opaque>`; `<opaque>` = Base64 (custom alphabet) ciphertext, decrypted with
`com.epic.util.h.g.n` → AES/CBC/PKCS5Padding decrypt (key = ASCII bytes "9FE3C86E96C5CC9C",
IV = 0xD2ADBEEF×4) → string → `new URL(...)`. So MDM / QR links carry an *encrypted* econfig URL.

Cache file: **`<configuration-guid>.econfig` in `Context.getFilesDir`** (`v1.l` → private
`v1.d0` via `openFileOutput(name, MODE_PRIVATE)`; deletion in `v1.q` via `deleteFile(guid+".econfig")`).
New configs are stored **plaintext inner XML**; legacy encrypted files are transparently migrated
(`v1.X`: parse container, decrypt (empty password → Salt-as-password path), rewrite decrypted,
keep filename). Freshness: `Generated` timestamp compared against stored date (`v1.u/C0`), also
`l2.C0` logs "up to date". Password prompt state machine in `l2.G0` (MDM vs AUTOMATIC_UPDATE
suppression); per-GUID metadata in SharedPreferences (`v0`/"configuration" pref), MDM-only GUID
`b2` / GUID-conflict `z1` exceptions.

### 4. MDM configuration keys (confirmed)

Read in `com.epic.haiku.android.authentication.p1`:

- `p1.tc(boolean)` — `getSystemService("restrictions")` → `RestrictionsManager.getApplicationRestrictions`:
 - **`EpicMobileConfigurationURL` (String)** — non-empty string → `p1.nb(v, false)` → `k2.kC(...)`
 → `k2` no-UI fragment (`urlHandlerNoUI`) with args `econfig_url`, **`econfig_is_from_mdm=true`**,
 `refresh_all_econfigs`; the value MUST be an `epic|cipe://handheld/config/<encrypted-url>` link
 (plain https://.econfig URLs are rejected by `k2$a.a` — resolves the pass-1 ambiguity: the MDM
 value is the encrypted `epic://handheld/config/...` form, not a raw.econfig link).
 - **`restrictions_pending` (Boolean)** — when true, MDM push hasn't settled; shows a "pending"
 message and defers handling (prefs key `restrictions_pending`).
 - empty-restriction edge cases surface string resources (`yi`, `wf`) as user messages.
- Downstream flags: `econfig_is_from_mdm` propagates to `e10.c`-based install (`l2$z0(..., isMdm)`),
 `v1.l` refuses non-MDM overwrite of an MDM-managed GUID (`com.epic.haiku.android.configuration.b2`),
 and `v1.Y` maps config source to `com.epic.android.d` events (DOWNLOAD / MDM / OVERRIDE).

Other RestrictionsManager keys: none found — `EpicMobileConfigurationURL` and
`restrictions_pending` are the only keys read (xref; `readRestrictionsManager` string belongs
to unrelated code).

### 5. ServerInfo / capability model

`com.epic.communication.ServerInfo` — full field list:

| field | accessor |
|---|---|
| `_encryptionKey` (String) | `getEncryptionKey` |
| `_version` (String) | `getVersion`; compared with `versionIsAtLeast` via `com.epic.util.a0.b` |
| `_isAndroidSupported` (boolean) | `isAndroidSupported` |
| `_capabilities` (Set<String> of wire values) | `getAllCapabilities` (unmodifiable) |
| `_readOnly` (boolean) | `isReadOnly` |
| `_loginMessage` (String) | `getLoginMessage` |
| `_isUserLoggingEnabled` (boolean) | `getIsUserLoggingEnabled` |
| `_userHasEmbeddedHSAccess` (boolean) | `getUserHasEmbeddedHSAccess` |
| `_envID` (String, mutable) | `get/setEnvID` |
| `_loginWizardEnabled` (bool, mutable) | `get/setLoginWizardEnabled` |

`hasCapability(com.epic.util.Capability)` matches when the wire value `_value` is in the set **OR**
`_version >= Capability._version` (so version-named base capabilities like BASE_2014 are
auto-true on newer servers). Server version benchmarks: 7.8/2010, 7.9/2012, 8.1/2014, 8.2/2015,
8.3/Gasherbrum, 8.4/2018, 8.5/2018-Aug, 8.6/2018-Nov, 8.7/2019-Feb, 8.8/2019-May, 8.9/2019-Aug,
9.4/2020-Aug, 10.2.0/2022-May, 10.6.0/2023-Aug.

`com.epic.haiku.communication.model.HaikuCapability` — enum, 401 constants, each with a
**wire value string** (what the server puts in the capability set) and optional min-version. ~40 samples
(enum name = wirevalue):

```
LIMITED_ORDERING=HaikuOrdersECF FULL_ORDERING=FullOrders REFILL_REQUEST_BPA=RefillRequestBPA
IB_MED_ONLY_COMMANDS=RxOnlyCommand FULL_I18N=FullI18N_Android INBASKET_CUSTOM_COMMANDS=CUSTOMCOMMANDS
CAPABILITY_SIGN_AT_CLOSE_ENCOUNTER=SignAtCloseEncounter E4C_SCHEDULE_CAPABILITY=E4CSchedule
NOTIFICATION_CAPABILITY=PushNotifications ENHANCED_DEPT_SELECTION=ENHANCED_DEPT_SELECTION
AUTOMATIC_ENCOUNTER_SELECTION=AutomaticEncounterSelection INBASKET_LONG_COMMAND_ID=LongIBCommandID
DX_ASSOCIATION=DxAssociation AN_READ_ONLY_CHECKS=ANReadOnlyChecks
AN_RECORD_SELECTION_CHECKS=ANRecordSelection SUPPRESS_SIG_DURATION=DISPCALC
IB_CALL_ME=IBCallMeCapability INBASKET_PATIENT_CALLS=IBPatientCall
BADGE_MANAGER_WEBSERVICE=BadgeManagerWebService ENCOUNTER_CREATION_CAPABILITY=ECFEncounterCreationCapability
EVENT_FRAMEWORK_SUPPORT=EFNotificationsCapability MULTI_PARAMETER_PATIENT_LOOKUP=MultiParameterPatientLookup
```

Full name→wire-value list extracted to `/tmp/caplist.txt` during the pass (388 named wire values of
401 constants; 13 carry empty value or version-only semantics, e.g. NONE, BASE_20xx with
version-only match). `t60.a.c/e` gate Logout/KeepAlive ECF-vs-legacy on
`MOBILE_ECF_LOGOUT_CAPABILITY`; `t60.a.m` routes commands to JSON CommandWebAPI when the response
schema declares a web API override.
