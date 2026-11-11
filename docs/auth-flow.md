# Epic Haiku 11.9.2 (Android) — Authentication Architecture

Class and field names are as they appear in the shipped client (short single-letter package
names such as `t60`, `c10` are the client's own internal names; `L...;` forms are VM
type descriptors, reproduced verbatim).

## 1. Transport and connection layer

### 1.1 Base classes
- `com.epic.communication.d` — generic Command object: name, request object, response class,
 options (`com.epic.communication.r`: SuppressConnectionErrors, VerboseLogging, RetainOwner, …),
 timeout, `requiresAuthentication` flag, execution queue.
- `c10.d` — abstract "connection profile" (one per auth mode). Holds:
 - base `java.net.URL` (the Interconnect base URL of the environment, from `e10.g.r`),
 - a set of command names that are forced onto the WebAPI transport,
 - execution-info factory `r(c10.c)` which stamps every command with:
 - command endpoint base URL (`c10.c.d`),
 - the **session token** (from `EpicSettings.getSessionToken` → `c10.c.p` / `.h`),
 - debug id, forwarded-for IP (`c10.a` app-wide settings).
- `c10.c` — per-command execution info: base URL, path group `e`/`f`,
 relative-path override set by `.o(path)` (`f` field), session token (`p`/`h`),
 `Epic-External-Request` flag (`j/a`), `X-Forwarded-For` (`q/i`).
- `Lt60/a` — Haiku's concrete base connection profile. Static endpoint construction:
 legacy transport URL = `<base URL>/httplistener.ashx`. Chooses transport per command:
 - command name NOT in the profile's "WebAPI set" and no capability requirement → legacy Interconnect
 SOAP facade (`q60.d` + `com.epic.haiku.communication.internal.facade.FacadeServiceOperation`,
 transport layer name `"InterconnectFacade"`, SOAP 1.2 envelope, `application/soap+xml`,
 requests versioned `8.1`);
 - otherwise → `com.epic.communication.webservices.execution.CommandWebAPIServiceOperation`
 (transport layer `"MobileECF-WebAPI"`).
 - Also builds KeepAlive and Logout commands (see §5).

### 1.2 Concrete connection profiles
- `Lt60/i`, "OAuth2 / password profile") — `super(baseURL, {PreLogin, Authentication.Login,
 TerminateRemoteSession, SearchRecordsPreLogin})`; constructor `(e10.g environment, String clientSecret,
 String applicationId)`. Overrides two relative paths in `f`:
 - `Epic.Mobile.Core.Authentication.EpicOAuth2Login` → `internal/Auth/OAuth2/EPICOAUTH2LOGIN`
 - `Epic.Mobile.Core.CreateSession` → `api/epic/2017/Auth/OAuth2/CREATESESSION`
 - `b(info, urlConnection)`: attaches header `Authorization: <type> <token>` (field `t`, set via
 `v(token, type)`) — but only when environment auth mode is not `w`-marked (i.e. not OIDC)
 **and** the command name is in the set {`Epic.Mobile.Core.Authentication.EpicOAuth2Login`,
 `Epic.Mobile.Core.CreateSession`} (set `A`, static).
 - *Uncertainty: exact gating logic — the branch condition also drops the header entirely when
 `environment.w` is false; w is an environment flag (likely "uses Authorization headers").*
 - `d(...)` creates the login SyntheticCommand `com.epic.haiku.communication.session.h` (wizard login);
 if the credential's current step is `Start` and auth mode is `OPENID_CONNECT`, passes the stored
 ID token (`w(idToken)` field `x` → `h.s(idToken)`).
- `Lcom/epic/login/oauth2/OAuth2ClientInfo` — second profile for the client-side OIDC/OAuth2
 login, `super(baseURL, {Epic.Mobile.Core.CreateSession})`. Builds its own login/CreateSession command
 (§3.3) and also sets `Authorization: <type> <token>` (`x(token, type)` → fields `r`,`q`).
- `Lt60/e`, "OIDC profile") — WebAPI set {`InitiateOpenIDConnectLogin`, `PreLogin`, `Login`,
 `Authentication.Login`, `TerminateRemoteSession`}; attaches `Authorization: t` (from `v(token,type)`)
 **only when `environment.w`** (only for OIDC-class environments). Login command is
 `com.epic.haiku.communication.session.d` (non-wizard variant).
- Environment: `e10.g` — named server/environment; address (`g`),
 settings `HashMap` (keys like `LOGIN.REMEMBER_LAST_USER`, `HEADER.DISASSOCIATE_ON_LOGOUT`,
 `DESKTOP.HYPERSPACE_URL`), auth-type enum `e10.g$a` (values seen: `DEFAULT`, `OAUTH2`,
 `OPENID_CONNECT`; `q` = auth type, `f` = GUID, `r` = base URL, `w` = unknown but OIDC-specific,
 `z` = likely "Epic-issued OAuth2 (server-side OIDC)" flag).

### 1.3 WebAPI transport (`CommandWebServiceOperation` / `CommandWebAPIServiceOperation`)
For every WebAPI command:
- URL: `<base URL>` + relative path; default path is
 `internal/{groupA}/{groupB}/{commandName}` from `getRequestRelativePath` =
 `String.format("internal/%s/%s/%s", info.e, info.f, command.name)`, unless `.o(path)` overrode it.
- **Method: POST**.
- Headers:
 - `Authorization: <type> <token>` — only added by the connection profiles above (OAuth2/OIDC commands).
 - `Interconnect_SessionToken: <session token>` — added by `connectionFromCommand` when
 `command.getRequiresAuthentication` is true (this is the **post-login session token header**).
 - `Epic-Debug-ID: <id>` — if set.
 - `Accept: application/json` (or `multipart/mixed+json`).
 - `Content-Type: application/json; charset=utf-8` (or `multipart/mixed+json; boundary=<uuid>` for blob
 requests — first part is always the JSON body, additional parts carry `Content-Disposition: inline;
 name="<field>"`).
 - `Interconnect_ClientTime: <client timestamp>` (from `com.epic.util.u.a`).
 - `Epic-External-Request: true` — when the command is flagged external (`c10.c.a`).
 - `X-Forwarded-For: <ip>` — when set.
- Body: JSON serialization of the request object (`d10.c`), or multipart as above.
- Response: JSON into the response object (`d10.b`), multipart → JSON part 0 + named binary parts.
- Errors: HTTP 400/401/404/500 mapped to error codes; response "reason" strings
 `SESSION-EXPIRED`, `PATIENT-WORKSPACE-CLOSED`, `PATIENT-MERGED`, `MOBILE-INTENTIONAL-THROW`,
 `INVALID-CLIENT-TIME`, server fault code `INVALID-AUTH-TOKEN` → `NotAuthorized`.

### 1.4 Legacy transport (pre-ECF environments)
`q60.d` + `FacadeServiceOperation`: SOAP POST to `<base>/httplistener.ashx`,
request name split into namespace + trailing token (`t60.a.f` puts the command-version token `8.1`
and the URN-derived request name into `q60.f` execution info), requests marked `Unauthenticated`
when `requiresAuthentication=false`. KeepAlive/Logout fall back to this when the server lacks the
`MOBILE_ECF_LOGOUT_CAPABILITY` capability (§5).

## 2. Login sequence (ordered HTTP exchanges)

All ECF commands below are POSTs to the WebAPI transport with the §1.3 headers/body rules.
`preLoginSettings` come from Epic.Mobile.Core.PreLogin earlier in startup.

### Step 0 — PreLogin (startup, before any credentials)
- Command `Epic.Mobile.Core.PreLogin`, request body `{}` (`PreLoginCommand$Request` is empty).
- Response (`PreLoginCommand$Response`): `settings: HashMap` (contains `PublicKey` used to encrypt
 passwords) and `localeInfo`.
- Response also carries the `MOBILE_ECF_LOGOUT_CAPABILITY`-style capability list via `ECFServerInfo`
 (delivered later with login; PreLogin response carries general settings).

### Step 1 — Login orchestration (`com.epic.haiku.communication.session.h`
— SyntheticCommand spawned by `t60/i.d`; the "credential" `com.epic.communication.i0` carries
loginID, ID type, password chars, selected template/dept, current step, workflow, AuthToken,
optional `ECFOpenIDConnectCredential`)

Decision tree in `h.beginWork`:
1. If a password is present and the environment GUID + pre-login `PublicKey` fail the policy check
 (`t60/g.a(guid, publicKey)` returns `t60.f.FailClosed` → e.g. missing public key when required) →
 immediate fail (`Failed` ECFLogin$Response).
2. If server-side OAuth2 applies (`m`: fresh `Start` step + `environment.q == OAUTH2` + no
 biometric refresh token stored) → continue to (3), else jump to (5).
3. If `environment.z` → branch (4) else branch (3a):
 **3a. CreateSession-based OAuth2** (`h.q`): fire command
 `Epic.Mobile.Core.CreateSession` with empty request body (`OAuth2Login$Request{}`),
 response class `OAuth2Login$Response`. This POST goes to
 **`api/epic/2017/Auth/OAuth2/CREATESESSION`** with `Authorization: <type> <token>` attached
 (set by `t60/i.b`; at this point the OIDC/OAuth2 provider token was supplied by the login
 controller). Response: `OAuth2Info { loginToken, loginID }` → continue to (5) with that
 LoginToken + LoginID (step reset to `USERNAME`).
 **4. EpicOAuth2Login-based** (`h.p`): command
 `Epic.Mobile.Core.Authentication.EpicOAuth2Login`, empty request (`EpicOAuth2Login$Request{}`),
 response `EpicOAuth2Login$Response` → `OAuth2Info { loginToken, loginID }`; POST goes to
 **`internal/Auth/OAuth2/EPICOAUTH2LOGIN`** with the `Authorization` header as above → then (5).
5. **ECF wizard login** (`h.d`): command `Epic.Mobile.Core.Authentication.Login` with body
 `ECFLoginWizard$Request` (response `ECFLoginWizard$Response`); `requiresAuthentication=false`,
 options `SuppressConnectionErrors`, workflow record `HKU_LOGIN_COMMAND`.
 Body fields (`ECFLoginWizard$Request`):
 - `SelectedTemplateID`, `SelectedDeptID` — from credential
 - `Workflow` — enum number (`LoginWorkflowDefault=0`, `ChangeJob=1`, `ChangeDept=2`),
 - `SkipJobSelectionUpdateStatus`,
 - `CurrentStep` — key of `com.epic.communication.v0` (`Start`, `EpicNative2FA`, `ChangePassword`,
 `JobSelection`, `DepartmentSelection`, `CreateSession`, `LoginComplete`, `MobileConnectionLimit`),
 - `LoginToken` — set when (a) step != `Start` (continuing a wizard) or (b) the OAuth2 branch above
 supplied it or (c) a biometric/check-PIN token exists; value = OAuth2LoginToken or stored token,
 - `LoginStartInfo` — an `ECFLogin$Request` (see below),
 - `SetNewPassword` — `ECFSetNewPassword` when a password change is pending
 (password + confirm + `PublicKey`).
 `ECFLogin$Request` = `LoginStartInfo`:
 - `LoginID`, `LoginIDType` (int), `ClientTime` (Date),
 - `ClientSecret` — the string passed through `t60/i` constructor (`clientSecret`, likely an
 app/installation secret from Haiku configuration); *uncertainty: exact semantics — this is the
 only place the field name appears; it is always.e(this.b), i.e. the connection-profile field.*
 - `SelectedTemplateID`, (builder `.p(skipJobSelectionUpdateStatus)` on non-wizard path)
 - `EncryptedPassword`, `PasswordIV`, `EncryptedAESKey` — the password envelope, built when a
 password is present (exact construction below; password buffer zeroed afterwards),
 - `AuthToken` + derived `TokenType` — one of `ID` (OIDC ID token), `PIN`
 (`Epic2FA`/`LDAP` normalized to `PIN`), or default `BiometricRefresh` when none supplied;
 `RefreshToken` getter = the raw token string.
 - `SystemInfo`, `ApplicationInfo`(`ECFApplicationInfo(applicationId)`), `DeviceInfo`,
 `LocaleInfo`, `TimeZoneInfo`, `ClientConfiguration`(`ECFClientConfiguration(environment)`),
 `LoginNavigationInfo`(`ECFLoginNavigationInfo(workflow, step)`).
 - `OpenIDConnectCredential` — `ECFOpenIDConnectCredential{ sessionID, deviceID,
 authorizationCodeResponse: Map }` — attached only in OIDC mode (§4).
 The wizard may require multiple round trips; each iteration re-execs
 `Epic.Mobile.Core.Authentication.Login` with `CurrentStep` advanced
 (`EpicNative2FA` → asks for one more AuthToken; `ChangePassword` → `SetNewPassword`;
 `JobSelection`/`DepartmentSelection` → selection info filled).

### Step 2 — Successful login response (`ECFLoginWizard$Response` / `ECFLogin$Response`)
Fields relevant to session/token model:
- `Code` (`com.epic.communication.u0`: Success/Failed/UserPasswordExpired/Inactive/Disabled/…,
 legacy codes `LegacyLoginResultCode`), `CurrentStep`, `Prompts`, `Menus`,
- `UserInfo` (`ECFUserInfo`: userID, providerID, loginID, name, settings map, departments),
- `ContextInfo` (departments/job templates), `JobSelectionInfo`, `PasswordChangeRequirements`,
- `SessionInfo` (`ECFSessionInfo`: **`Token`** = Interconnect_SessionToken, `Timeout`),
- `ECFServerInfo` (server version, `EncryptionKey`, capabilities, readOnly, envID),
- **`RefreshToken` + `RefreshEnabled`** (the "native biometric token" — see §6.2),
- `TokenExpiration`, `OfflinePINExpiration`,
- `CheckPINToken` / `CheckPINAccessToken` + `CheckPINAccessTokenExpiration` (subsequent-auth/PIN),

- `EventNotificationServerURL/ClientID/AccessToken` (SignalR push),
- `VinzInfo` (`ECFVinzInfo` — "Vinz" cloud identity descriptors used by Nebula; §7),
- `EpicSettings` constructed client-side (`sessionToken`, user, departments, session timeout,
 login schedule).

### Step 3 — Session established
The Interconnect session token is taken from the login response (`EpicSettings.setSessionToken`),
and every subsequent authenticated command carries it in the `Interconnect_SessionToken` header
(`CommandWebServiceOperation.connectionFromCommand`, gated by `requiresAuthentication`).

### Non-wizard variant
`com.epic.haiku.communication.session.d` (used by `Lt60/e` OIDC profile) is identical but can send
plain `Epic.Mobile.Core.Login` (response `ECFLogin$Response`) instead of the wizard command when the
workflow doesn't need wizard steps.

## 3. SSO options

### 3.1 Server-side OIDC ("Epic OAuth2") — the `EpicOAuth2Login` / `CreateSession` branch
Covered in §2 Step 1 (3a/4). Summary:
- `POST internal/Auth/OAuth2/EPICOAUTH2LOGIN` (or `api/epic/2017/Auth/OAuth2/CREATESESSION`),
 empty JSON body, `Authorization: <type> <token>` header carries the token obtained out-of-band
 (e.g. from the facilitated/QR login or external IdP), returns `{OAuth2Info: {loginToken, loginID}}`.
- The loginToken then drives the ECF Login wizard (`LoginToken` in `ECFLoginWizard$Request`).

### 3.2 Client-side OIDC with Epic's embedded client (`com.epic.oauth2.*`)
A compact home-grown OIDC client plus an MSAL wrapper:
- `m$a` — redirect URIs: custom scheme `com.epic.haiku.oidc://callback`
 (`com.epic.rover.oidc://callback` for Rover), universal links
 `https://mobile.epic.com/haiku/OIDC/redirect/` and `https://mobile-test.epic.com/haiku/OIDC/redirect/`;
 all four are registered as possible callback targets.
- `com.epic.oauth2/i` — "MSALConfiguration" `{clientID, scope, authorityUrl, acr, loginHint,
 forcesReauthentication}`; produces an Azure AD claims parameter (`id_token.acrs.essential = acr`).
- `com.epic.oauth2/c` — "OIDC config" derived from MSAL config:
 `authorizationEndpoint = {authority}/oauth2/v2.0/authorize`,
 `tokenEndpoint = {authority}/oauth2/v2.0/token`, scope = `openid profile` + config scope,
 PKCE on by default, optional `clientSecret` (Basic auth on token calls), optional redirect override.
- `com.epic.oauth2/w` — token endpoint POST:
 - `grant_type=authorization_code` + `code` + `redirect_uri` + `code_verifier` (PKCE S256),
 optional `client_id` or `Authorization: Basic base64(clientID:clientSecret)`;
 - `grant_type=refresh_token` + `refresh_token` + optional `scope`;
 - `Content-Type: application/x-www-form-urlencoded`, optional extra query params merged.
- `com.epic.oauth2/u` — token result `{access_token, token_type, expires_in(→ local expiry,
 60s skew), refresh_token, id_token}`.
- `com.epic.oauth2/j` — MSAL (Azure AD) single-account manager (brokers via
 `broker_redirect_uri_registered`; tenant extracted from authority path, defaults `common`);
 used when server config supplies `MicrosoftAuth` details.
- `OAuth2PrivateBrowserActivity` (`com.epic.login.oauth2`) — incognito Custom Tab that hosts the
 authorization flow + `OAuth2LoginTimeoutWorker` (kills the task after timeout).

### 3.3 Client-side OIDC — command-level flow (`Epic.Mobile.Core.Authentication.InitiateOpenIDConnectLogin`)
- Request `InitiateOpenIDConnectLogin$Request` →
 `{ DeviceID, ClientID, AuthorizationURL, UseNonProductionAssociatedDomain, ApplicationID }`
 (also a 1-arg variant `{DeviceID}` = "continue with known device").
- Response → `{ SessionID, DeviceID, AuthorizationURL, MicrosoftAuth, ClientOIDC }`.
 - `MicrosoftAuth` → `{ ClientID, Scope, Authority, ACR, LoginHint }` — switches the client to
 MSAL instead of the browser flow.
 - `ClientOIDC=true` → client performs the OIDC dance itself (§3.2).
- Browser/CustomTab/MSAL returns the code response; it is packed into
 `ECFOpenIDConnectCredential{sessionID, deviceID, authorizationCodeResponse}` and sent inside
 `Epic.Mobile.Core.Login` / `Epic.Mobile.Core.Authentication.Login`
 (`ECFLogin$Request.OpenIDConnectCredential`) — the server completes the code exchange when the
 code response handling stays server-side.
- For OpenID_CONNECT environments `OAuth2ClientInfo.d` uses
 `Epic.Mobile.Core.Login` with `ECFLogin$Request.AuthToken = AuthToken(idToken, TokenType=ID)`.
- The OIDC ID token may also be cached on the profile (`t60/i.w(idToken)`) for subsequent wizard
 starts.

### 3.4 JWT-bearer grant (Nebula/"Vinz" identity)
`com.epic.observatory.nebula.VinzAuthenticationManager`):
- `POST {vinzUrl}`, `Content-Type: application/x-www-form-urlencoded`, read timeout 20s,
 body: `&grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer&client_id={clientId}&assertion={ID-token}[&scope={scopes joined with +}]`
 (note leading `&`).
- Response `{access_token, expires_in}` (stored with 20s safety skew); errors `{error}`.
- The assertion is the ID token previously issued by the identity provider (Epic Nebula
 `Epic.Mobile.Core.Authentication.Nebula.IssueIDToken` or the OIDC id_token); the resulting
 access token authenticates Nebula (Epic cloud) API calls, tracked by
 `com.epic.login.nebula.NebulaTokenManager` (token store, per-"Vinz identity" auth managers).

### 3.5 SAML enterprise SSO
Strings and classes seen: `Epic.Mobile.Core.Authentication.GetSAMLAuthSettings`,
`Epic.Mobile.Core.Authentication.ValidateSAMLAuth(.Request/.Response)`,
`SAMLAuthDeviceFragment` / `WebFormAuthType.SAML` / `AuthenticationDeviceType.SAML`,
SAML 2.0 metadata URNs, `urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect`,
`epic://SAML/redirect` deep link (the client), built-in SAML Response/Assertion templates
in the mock-IdP test harness (`com.epic.haiku.android.infrastructure.activities.mockidpauthenticator.*`).
Flow (inferred, *uncertain details*): server provides SAML auth settings → Haiku opens the IdP web
form (SAMLAuthDeviceFragment WebView) → IdP posts the SAML Response back to Haiku through the
`epic://SAML/redirect` deep link → client sends it to
`Epic.Mobile.Core.Authentication.ValidateSAMLAuth` → validated SAML user is then logged in via the
normal ECF Login command. **Not fully traced: exact body of ValidateSAMLAuth**
(the request type `com.epic.authentication.command.ValidateSAMLAuthRequest` was not traced).

### 3.6 QR / badge "mobile facilitated login" (MFL)
Command pair (`com.epic.haiku.android.hsmobile.*`; command names in
`HSMobileFacilitatedLoginCommands.kt`), executed toward the Hyperspace/workstation side
(Epic.Mobile.Workstation namespace):
1. `Epic.Mobile.Workstation.GetMFLPreLaunchToken`
 request `{ codeChallenge, codeChallengeMethod }` (PKCE S256) →
 response `{ preLaunchToken, errorMessage }`.
 Strings: "Loading pre-launch token for mobile facilitated login",
 "No launch token received…", "Sending pre-launch token…".
2. `Epic.Mobile.Workstation.GetMFLAuthCode`
 request `{ launchToken, codeVerifier }` → response `{ authCode, errorMessage }`.
3. The authCode is then used in the OAuth2/OIDC login path (strings "Sending/Loading
 authentication code for mobile facilitated login"); *uncertain: whether it rides the
 `EpicOAuth2Login` Authorization path or feeds `ECFOpenIDConnectCredential`.*
- Hyperspace side uses `com.epic.mobiledesktop.communication.MobileDesktopSecurityOfficer.
 MobileFacilitatedLoginData` (protobuf/$serializer), `CancelMFL`,
 and the Haiku "Mobile Desktop" web container (`com.epic.mobiledesktop.m`) stamps the header/cookie
 `X-EpicMFL-CanAttempt=1; Max-Age=30; SameSite=Strict;` allowing an MFL attempt for 30 s.
- Cancellation string: "Mobile facilitated login cancelled by Hyperspace".

## 4. Token model (summary)

| Token | Obtained from | Used as | Refresh |
|---|---|---|---|
| OAuth2 `loginToken` (Interconnect "OAuth2 login token") | `EpicOAuth2Login`/`CreateSession` response `OAuth2Info` | `ECFLoginWizard$Request.LoginToken` to drive ECF Login | — (single-use) |
| **Interconnect SessionToken** | `ECFLogin$Response.SessionInfo.Token` | `Interconnect_SessionToken` request header on authenticated WebAPI commands | re-login / biometric refresh token |
| OIDC `id_token` | IdP (browser/MSAL) or `OAuth2Info` | `ECFLogin$Request.AuthToken` (TokenType=ID); also Vinz JWT-bearer assertion | — |
| OIDC `access_token`/`refresh_token` | `{authority}/oauth2/v2.0/token` (PKCE or refresh grant) | `Authorization: Bearer` on EpicOAuth2Login/CreateSession; refresh grant on expiry | refresh_token grant |
| **Biometric "native" refresh token** | `ECFLogin$Response.RefreshToken` (+`RefreshEnabled`) | stored encrypted; sent as `ECFLogin$Request.AuthToken` (TokenType=BiometricRefresh / RefreshToken field) for subsequent auth | server issues new one with each ECF Login ("Login response did not include a new native biometric token; refreshEnabled: ") |
| Check-PIN token | `ECFLogin$Response.CheckPINToken(+Expiration)` | CheckPIN subsequent-auth command | server-supplied expiration |
| Nebula access token | Vinz JWT-bearer exchange | Nebula cloud API | re-exchange (20s skew) |

Session expiry → server fault `SESSION-EXPIRED` (CommandWebAPIServiceOperation error mapping)
and `INVALID-AUTH-TOKEN` → `NotAuthorized`; the client then surfaces re-authentication.

## 5. Keepalive / logout / remote session termination

- **KeepAlive** (`Lt60/a.c`): if `ServerInfo` has capability `MOBILE_ECF_LOGOUT_CAPABILITY`:
 command `Epic.Mobile.Core.KeepAlive` with `KeepAliveRequest`→`KeepAliveResponse`;
 legacy fallback: SOAP command URN `urn:epicsystems-com:EpicOnHand.2010.Services.Login.KeepAlive`
 via `httplistener.ashx`. Option `SuppressConnectionErrors`. Client side drivers: AutoLogoutService
 (`com.epic.haiku.android.infrastructure.service`), `LockingWorkflowKeepAliveSender/Caller`
 (locking workflows), `keepAlivePref` timer settings.
- **Logout** (`Lt60/a.e`): with capability → `Epic.Mobile.Core.Logout` (`ECFLogoutRequest`→
 `LogoutResponse`); legacy fallback: URN
 `urn:epicsystems-com:EpicOnHand.2009-08.Services.Login.Logout` with `LegacyLogoutRequest`.
 Both with `SuppressConnectionErrors` + `VerboseLogging`. Extensive client logout plumbing
 (`AutoLogoutService`, embedded-Hyperspace `LogoutService.Request`, "Device dissociation during
 logout", notification dissociation `HEADER.DISASSOCIATE_ON_LOGOUT` setting).
- **TerminateRemoteSession**: `Epic.Mobile.Core.TerminateRemoteSession` with
 `ECFTerminateRemoteSession$Request { applicationID, deviceID, key, token }` →
 `Response { token, sessions[], errorMessage }`. UI: `JobSelectionSharedViewModel.
 createAndSendTerminateRemoteSessionCommand`, `SessionTerminationDialogFragment`
 (job-selection step tells the server to end the other/remote session).
- `SearchRecordsPreLogin` (`Epic.Mobile.Core.SearchRecordsPreLogin`) — pre-login record search,
 unauthenticated WebAPI command (part of `t60/i` WebAPI set).

## 6. Credential storage

### 6.1 Biometric unlock (`com.epic.login.biometric`,
- Key pair `HaikuFingerprintKeyPair` generated in **AndroidKeyStore**: RSA, digests
 `SHA-256`/`SHA-512`, padding `OAEPPadding`, `setUserAuthenticationRequired(true)`
 (`com.epic.login.biometric.c`); cipher transformation string `RSA/ECB/OAEPPadding`
 with `OAEPParameterSpec(SHA-256, MGF1-SHA1)`.
- BiometricPrompt (`androidx.biometric.q.f`) is invoked with a **CryptoObject** wrapping that
 cipher; `KeyPermanentlyInvalidatedException` → key re-enrolled, all stored tokens wiped
 ("Android Keystore reported a permanently invalidated biometric key during enrollment").
- Token wrapper `pa0.f` stores {native token (c/d), OAuth refresh token (b/e), token type (b),
 expiration (a)}; on successful prompt the stored token is decrypted and used as
 `new AuthToken(token, AuthToken$a.BiometricRefresh)` and handed to the credential provider
 delegate (`mCredentialProviderDelegate.Cv(...)`, `ha0.e`) → login continues with the biometric
 refresh token.
- Persistence: plain **SharedPreferences file `"biometric_store"`** (`com.epic.login.biometric.d`)
 with per-(environmentGuid, authType, loginId) keys:
 - `biometric_store:token:{envGuid}:{loginId}` (RSA-encrypted token wrapper — *uncertain: exact
 container; the base class `pa0.e` applies the keystore crypto before storage*),
 - `biometric_store:enabled:{envGuid}:{loginId}` (client preference),
 - `biometric_store:permitted:{envGuid}:{loginId}` (server permission, written from ECF response),
 - `biometric_store:username:{envGuid}:{loginId}`,
 - `biometric_store:type:{envGuid}:{loginId}` (biometric type, `ja0.j`).
- Server-side opt-in arrives in the login response (`RefreshEnabled` + `Permitted`); bulk invalidation
 helper `d$t.s(context, reason)` removes all keys with the `biometric_store:` prefix.
- No EncryptedSharedPreferences / SQLCipher involvement found in the login/biometric path.
 (SQLCipher `net.zetetic.database.sqlcipher.*` is present in the app for the offline cache,
 and Microsoft identity-common/adal storage for MSAL account state — `SSOStateSerializer`.)

### 6.2 Native refresh ("biometric login token") lifecycle
- On a normal password/OIDC login success, `ECFLogin$Response.RefreshToken`/`RefreshEnabled` carry a
 new "native biometric token"; client logs "Saving biometric login token; hasNativeToken…,
 hasOAuthRefreshToken…, hasTokenType…, secondsUntilExpiration…" and stores it (§6.1).
- Next login uses it as `AuthToken(tokenType=BiometricRefresh)` in the ECF Login request (or drives
 CheckPIN re-auth); if the response carries another refresh token it replaces the stored value.

## 7. Notes for reimplementation
- Two-step token acquisition is the dominant pattern: get an OAuth2 loginToken (or OIDC credential)
 → run the ECF Login wizard → obtain the Interconnect SessionToken → use it in
 `Interconnect_SessionToken` for everything.
- The `Authorization: Bearer` header appears only for the three OAuth2 bootstrap commands
 (`EpicOAuth2Login`, `CreateSession`) — everything else is the session-token header.
- `Epic-Client-Time`/`Interconnect_ClientTime` sync and `INVALID-CLIENT-TIME` faults mean clock skew
 matters (server validates client time).
- Passwords are never sent in clear: hybrid RSA/AES encryption against the PreLogin `PublicKey`
 (exact construction below); buffer zeroed in-memory.
- **Password envelope** (`EncryptedPassword` / `PasswordIV` / `EncryptedAESKey`, all Base64):
 when the server supplies an RSA public key (PEM) in the PreLogin response, a random
 16-byte AES-128 key and 16-byte IV are generated (`SecureRandom`);
 `EncryptedPassword` = Base64(AES-128-CBC/PKCS5Padding(UTF-8(password), key, IV));
 `EncryptedAESKey` = Base64(RSA-OAEP-with-SHA-1-and-MGF1-SHA1(key, server public key))
 (`RSA/ECB/OAEPwithSHA-1andMGF1Padding`; the public key is parsed from its PEM
 modulus/exponent); `PasswordIV` = Base64(IV) — the IV travels **in the clear**, only the
 AES key is RSA-wrapped. Legacy fallback when no public key is present: `EncryptedPassword` =
 Base64(AES-128-CBC/PKCS5Padding(UTF-8(password), key = 16 zero bytes, IV = 16 zero bytes));
 `PasswordIV`/`EncryptedAESKey` are omitted — the same fixed-key AES-128-CBC construction
 as the legacy "Epic Scramble" family used for econfig payloads.
- The MFL and OIDC flows both reduce to PKCE authorization-code exchanges; the redirect URIs to
 register for a clone are `com.epic.haiku.oidc://callback` and the mobile.epic.com universal link.

## Uncertainty list
1. Exact semantics and source of `clientSecret` (field `r` in `t60/i`, sent as `ECFLogin$Request.
 ClientSecret`) — presumably a Haiku app secret provisioned in configuration; not traced to its
 configuration source.
2. `e10.g.q`/`w`/`z` exact enum/flag meanings (auth mode OAUTH2 vs OPENID_CONNECT etc.).
3. SAML `ValidateSAMLAuth` request body and the exact handoff from `epic://SAML/redirect`.
4. Consumption point of the MFL `GetMFLAuthCode` result (authCode) — which command/path it feeds.
5. The precise container of `biometric_store` tokens (custom serializer vs EncryptedSharedPreferences).
6. `t60/others` (e.g. `t60/o`, `t60/p`) and error-code enum `com.epic.communication.j` full mapping.
7. Which header/cookie actually triggers MFL attempt on the workstation side
 (`X-EpicMFL-CanAttempt` cookie was observed in the Haiku web container constructor).
8. Wizard intermediate step payloads (EpicNative2FA prompt schema, OTP SMS/SignalR details) — the
 `QBA`/`OTPSMSDetails`/`OTPSignalRDetails` schemas were not traced.
