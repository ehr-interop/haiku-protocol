# Epic Haiku 11.9.2 (Android) — TLS/pinning, embedded Hyperspace WebView, realtime/push channels

Class and field names are as they appear in the shipped client; short single-letter
package names such as `md0`, `b20`, `ec0` are the client's own internal names, and
`L...;` forms are VM type descriptors. Original Kotlin package names are recoverable
from the client's Kotlin metadata / stack-trace strings and are given where known.

---

## 1. Certificate pinning & TLS hardening

### Verdict: no certificate pinning detected

* The only TLS wrapper Epic installs is
 `com/epic/communication/webservices/execution/j`,
 constructed in `com.epic.communication.webservices.execution.CommandWebServiceOperation`
 around an `HttpsURLConnection`:

 ```java
 ((HttpsURLConnection) url.openConnection)
.setSSLSocketFactory(new j(conn.getSSLSocketFactory, new CommandWebServiceOperation$a(this)));
 ```

 `j` does **no pin validation**. It:
 * forces protocol list to TLSv1.3 + TLSv1.2 (from `SSLContext.getDefault.getDefaultSSLParameters`),
 * attaches a `HandshakeCompletedListener` (`execution/i` -> `j.c`) that throws
 `SSLException("<proto> not supported")` if the negotiated protocol is SSLv3 or TLSv1.1,
 * throws `UnsupportedOperationException("Clear text communication is not supported")` if the
 underlying socket is not an `SSLSocket`.
* The underlying `SSLSocketFactory` (`j.a`, labelled `"original"`) comes from a fresh
 `SSLContext.getInstance(<default protocols>)` with `init(null, null, null)` — i.e. the **platform
 default trust store / CA set**. No `TrustManager` injection, no custom `HostnameVerifier`
 (all `setHostnameVerifier` refs belong to okhttp3 `OkHttpClient$Builder`).
* okhttp3 is bundled and its `CertificatePinner` exists, but a search for
 `Lokhttp3/CertificatePinner;` references finds **zero references from Epic
 classes** — only okhttp internals (`RealConnection`, `OkHttpClient$Builder`). No
 `CertificatePinner$Builder.add(...)` call sites, and no string resembling a pinned hash ever
 appears in app code (`sha256/` occurs only in okhttp's own error message).
* The `trustedCertificatesPem` strings come from `io/opentelemetry/exporter/internal/TlsUtil`
 — unrelated to Epic.
* No trust-all / accept-all `X509TrustManager` was found in Epic code (no `checkServerTrusted`
 overrides outside libraries; scandit's `AllowExpiredCertificatesTrustManager` is confined to the
 Scandit barcode SDK).

### Header-level transport requirements (CommandWebServiceOperation)

* `POST` to endpoint; if connection is not HTTPS -> `finishWithSSLError(new SSLException("Https is required"))`
 when `vb0.d.a.o` (config flag) is true.
* Auth headers used by the command channel: `Interconnect_SessionToken` (session token when the
 command `requiresAuthentication`), `Accept`, `Content-Type`, `Interconnect_ClientTime`,
 `Epic-External-Request: true` (external requests), `X-Forwarded-For` (connection-provided client hint).

### WebView TLS handling

* `com.epic.hsmobileappcore.webview.window.WebViewWindowActivity$b.onReceivedSslError`
: **`handler.cancel`** — WebView TLS errors fail closed,
 the error is surfaced to `webResourceErrorLogger`/error UI.

### Uncertainties

* Certificate pinning could in principle be enforced server-side via Interconnect (`Epic Secure
 Configuration`), but no client-side pin structure exists in this build. If pinning exists it's
 delivered via server config interpreted natively — not found.
* `getInternalDebugTrustManagers` (a client string) has no method references —
 dangling okhttp3-platform string, not Epic.

---

## 2. Embedded Hyperspace WebView (`embeddedhyperspacecore`, `hsmobileappcore`)

The newer Haiku UI renders inside a system WebView loading **Hyperspace's server-hosted web app**
(EHS = "Embedded Hyperspace"; HSWeb). Key classes (internal names, original Kotlin names where known):

| DEX name | Original name |
|---|---|
| `b20/d0` | `com.epic.embeddedhyperspacecore.SingletonEmbeddedHyperspaceWebView` |
| `b20/p` (+`b20/a`) | `com.epic.embeddedhyperspacecore.EmbeddedHyperspaceFragment` (+ views) |
| `a80/c` | HSWeb WebView transport wrapper |
| `c20/i`, `c20/f` | `com.epic.embeddedhyperspacecore.communication.EmbeddedTransmissionSchemas` / dispatcher |
| `d80/b`, `d80/c` | `com.epic.hsmobilecore.model.communication.Transmission` + type enum |
| `f80/b`-based | `com.epic.hsmobileappcore.webview.HyperspaceAppWebView` |

### 2.1 Session authentication — PKCE token handoff, not cookies-first

Commands (Interconnect command channel `com.epic.communication.d`, both):

1. **`Epic.Mobile.EHS.ObtainHSCredentials`** (`com/epic/embeddedhyperspacecore/commands/ObtainEmbeddedHyperspaceCredentials`):
 * Request: `{ CodeChallenge }` where the challenge comes from `com.epic.oauth2.n`
: `codeVerifier` = 48-char `SecureRandom` Base64 string,
 `CodeChallenge = Base64URL(SHA-256(verifier))` — standard PKCE S256; no method field
 (server assumes S256).
 * Response: `{ LoginToken, PerimeterAuthenticationToken, HSUrl, Ticket }`.
2. **Credential assembly** (`b20/d0` ~line 2560):
 ```
 authValue = LoginToken + "|" + installationID + "|" + codeVerifier
 hsWebUrl = HSUrl + "/AutoLogin.aspx"
 loginTicket = Ticket
 perimeterToken = PerimeterAuthenticationToken
 ```
 (`installationID` from `vb0.d.a.d`). URL override paths exist for Epic-internal testing via
 SharedPreferences `prefListEHSUrlConfig` (`environment|elsa|perimeter|custom`, `prefEHSCustomUrl`,
 pinned test hosts `https://hsw-fs-test.epic.com/ElsaSDB`, `https://prj271932.epic.com`).
3. **WebView load** (`b20/d0` ~line 2324): `webView.loadUrl(hsWebUrl, headers)` where
 ```java
 headers = { "X-Login-Token": authValue, "X-Initialization-Ticket": loginTicket }
 ```
 The perimeter token is set as a cookie: `CookieManager.setCookie(hsWebUrl, perimeterToken)`
 (see `b20/d0` ~line 2761) — used when the Haiku server sits behind an Epic Perimeter
 (web-auth proxy) — uncertain on exact cookie name (string held in `PerimeterAuthenticationToken`).
 So: authentication = one-time server-issued **LoginToken + Ticket delivered via HTTP headers**
 (PKCE-verified server-side), plus an optional perimeter cookie; subsequent page loads ride on
 whatever server session `AutoLogin.aspx` establishes (cookies handled by `CookieManager`,
 isolated per profile when `MULTI_PROFILE` is supported — `HyperspaceAppWebView.setWebProfile`).
4. **Activity launch ticket** for opening a specific Hyperspace screen:
 **`Epic.Mobile.EHS.GetTicket`** (`GetActivityLaunchTicket`):
 Request `{ ViewParams: Map, ActivityDescriptor, ActivityID }` -> Response `{ Ticket }`.

### 2.2 Native <-> web bridge — `Transmission` over WebMessageListener

* Mechanism: androidx.webkit **`WebView.addWebMessageListener`** (`bc/e` wraps WebViewCompat;
 gated on feature string `"WEB_MESSAGE_LISTENER"` i.e. `WebViewFeature.isSupported`).
 JS object name: **`__zCommunicator`**;
 registered per-origin (`a80/c.p`), allowed origin rule = `<scheme>://<authority>` of the
 HSWeb base URL only — i.e. only the hospital's Hyperspace origin can call it.
 (`a80/c` = transport; note no legacy `addJavascriptInterface` fallback was found in this wrapper —
 if the WebView/feature is unsupported the bridge is absent, no postMessage fallback.)
* Message schema (`d80/b`, Kotlinx-serialization descriptor
 `com.epic.hsmobilecore.model.communication.Transmission`):
 ```json
 { "type": "Request|Response|Event", // enum d80.c
 "name": "<service name>",
 "id": "<correlation id, optional>",
 "data": <JsonElement, optional> }
 ```
 Log tag `HSWEB_JSON`; dispatch in `a80/c.j`: `Request` -> service handler, `Event` -> broadcast,
 `Response` -> resumes the pending Kotlin coroutine with the matching `id`.
* Known service names (`c20/i` subclasses, source `EmbeddedTransmissionSchemas`):
 `ActivityMessageService`, `CloseViewsService` (Request: viewIDs, forceClose, noSave),
 `ConnectionLostService`, `DeactivateViewService`, `HeartbeatService`, `OpenActivityService`,
 `ViewActivatedService`, `ViewClosedService`.
 Dispatcher `c20/f` maps `name` -> handler (`knownTypes` set built from `c20.f$b` enum entries),
 heartbeats every 5 s (`c20.f.r = Duration(5000 ms)`); "ActivityActionContext" per-activity state.
* Popup windows: `window.open` handled by `com.epic.hsmobileappcore.webview.window.WebViewWindowActivity`
 via a `HyperspaceWindowManager` (`b`): UUID `HyperspaceWindowManager.window_id_key` ->
 `Message.obj = WebView.WebViewTransport`, flag `HyperspaceWindowManager.secure_window_key`
 (default true -> `FLAG_SECURE` 8192). Log tag `HYPERSPACE_WINDOW_MANAGER`.
* WebView settings on that window: JS enabled, DOM storage + DB enabled,
 zoom built-in, geolocation enabled, `setAllowFileAccess(false)`, media autoplay allowed,
 multi-window enabled, `setWebContentsDebuggingEnabled(false)`.
* A `webOriginConsentStore` (per-origin feature consent) and `webResourceErrorLogger` are injected
 (Hilt) into the window activity.
* ClientBridge example from shipped assets (data, not executed):
 `assets/ConsentsResources/MobileConsents/Bridge/ClientBridge.js` defines message types
 (`Signature`, `SignatureRemoved`, `Zoom`, `DocumentManagerOnline`,...) forwarded to the client via
 the bridge; `assets/AndroidWebViewImageLoaderScript.js` injects image handling into pages.

### 2.3 Related web-launch flows

* **J2H (Jump-to-Hyperspace)** — `com/epic/haiku/android/jumpToHyperspace/J2HButton`,
 `Epic.Mobile.J2H.OpenHyperspaceActivity` command, broadcast `ACTION_JUMP_TO_HYPERSPACE`,
 pending listener type `com.epic.haiku.android.jumpToHyperspace.J2HPendingListener`.
* **Hyperspace Mobile Facilitated Login (MFL)** — `com/epic/haiku/android/hsmobile/GetMFLPreLaunchToken`
 (Request `{ codeChallenge, codeChallengeMethod="S256" }`) and `GetMFLAuthCode`
 (Request `{ launchToken, codeVerifier }` -> Response `{ authCode, errorMessage }`): again PKCE,
 used by `HyperspaceMobileLauncher` to hand Haiku's session to the standalone Hyperspace mobile.
* **FD Web session exchange** — `com/mmodal/mobile/MMToolbar$7` `POST`s
 `{ "accessToken":..., "impersonateSettingsOid":..., "source": "Mobile-SDK", "target":... }`
 to `<fdWebServer>/api/v1/session/exchange-token` and opens the `location` header URL in
 `FDWebActivity`. This is the dictation/Flow-Designer web side module (MModal), not the main
 Hyperspace path — worth documenting because it is the only literal "exchange-token" endpoint.
* `epic://SAML/redirect` — SAML SSO redirect capture during authentication
 (`GetSAMLAuthSettings`/`ValidateSAMLAuth` commands, `SAMLAuthDeviceFragment`, `webFormBaseURL`).

---

## 3. Realtime & push channels

### 3.1 SignalR client (bundled Epic-forked `microsoft.aspnet.signalr` Java client)

* Connection manager `md0/c`:
 wraps `microsoft.aspnet.signalr.client.hubs.HubConnection`, `HubProxy`, Gson, main-thread handler,
 exponential-backoff reconnect (`md0/b`), `KeepSessionAlive` handling.
* **Two "frameworks"** (`microsoft/aspnet/signalr/client/framework/*`); enum `SignalRFramework {ENv1=0, ENWS=1}`,
 chosen either by intent extra `SignalRFramework` or by whether a server URL is configured):

 | | **ENv1** (legacy) | **ENWS** (newer) |
 |---|---|---|
 | URL | `<base>/notificationservices/signalr` | `<base>/eventnotificationservice` |
 | Hub | `EventServiceHub` | `EventNotificationHub` |
 | subscribe | `SubscribeTopic(key, subkey, bool, cursor)` | `DynamicSubscribe(DynamicSubscribeRequest)` |
 | unsubscribe | `Unsubscribe(k, subkey)` | `DynamicUnsubscribe` |
 | publish | `Publish(keys[], type, data)` | `DynamicPublish` |
 | transport | `AutomaticTransport` (negotiates websocket/long-poll/SSE) | **`WebsocketTransport` only** |
 | cleanup/KeepSessionAlive | unsupported | supported |

 URL built by `ENv1Framework.constructURL` (the `/notificationservices/signalr` string lives there).
 `HubConnection(url, "ClientID=<eventNotificationClientID>", useDefault:true, logger, framework)`
 — the client ID is passed as the SignalR connection **querystring**.
* **Authentication** (`md0/c.G`):
 * If an OAuth access token is available: header `Authorization: Bearer <accessToken>`, hub name from
 framework; log "authentication successful using OAUTH2".
 * Otherwise (session-token mode): after `connected`, invoke hub method
 `SetConnectionParameters` with param `{"session-token": {"token": <Interconnect session token>}}`.
* Heartbeat: `SetConnectionParameters` result + periodic hub `KeepSessionAlive` (300 s timer `md0/b`,
 and explicit re-alive on network change — `CONNECTIVITY_CHANGE` receiver triggers reconnect).
* **Server->client callbacks** (`md0/d`, implemented by `...$ServerCallbacks`):
 * `newSubscribedEvent(Event[], int, String)` — batched events after subscribe,
 * `onEventReceived(ClientEvent)` — live event,
 * `resubscribe`, `error(String)`.
 Event model `com/epic/signalr/Event`:
 `Type`, `Data: Map`, `EventCursor`, `SubscriberIDs`, `TimestampInUtc`, `ID = "<key>-<subkey>"`.
 Cursor ordering enforced (`Event rejected. Old cursor:...`).
 Error sentinel: `SUB_DUPE` (duplicate subscribe, suppressed), `INVALID-SESSION-TOKEN` (session
 expired -> re-login notification), `ETIMEDOUT` (retry with jittered backoff).
 Subscription key = `md0/e(key, subkey)` pair ("topic" naming in push code: `EFTopic`).

### 3.2 In-app realtime service (Haiku foreground)

* `com/epic/haiku/android/signalR/ICEventNotificationService` (a bound `android.app.Service`):
 `onBind` reads extras `SessionToken, ServerURL, EventNotificationClientID,
 EventNotificationAccessToken, SignalRFramework`, then connects. Subscriptions via wrapped
 hub `Subscribe(key, subkey, bool, callback)`; server URL fallback derived from the connection
 URL by stripping `httplistener.ashx`. `SharedSessionICEventNotificationService` = shared-session
 variant with a 3 s unbind grace period.
* Payload consumers are arbitrary `f` subclasses (`com.epic.haiku.android.signalR.f` = subscription
 wrapper holding clientID/accessToken); events dispatched per cursor bump.

### 3.3 Push notifications — FCM primary, SignalR fallback

* **Google FCM is the primary channel.**
 * `com.epic.pushnotifications.PushNotificationsFcmListenerService extends FirebaseMessagingService`:
 rejects messages from unknown senders
 (`com.epic.pushnotifications.d.c` sender map), logs payload, forwards to router `ec0.e`/`ec0.d$c`
 (internal name only; log strings identify it as **`FcmNotificationRouter`** — its original class name
 is not preserved in the shipped build): decrypts encrypted FCM title/message
 (`"Decryption of encrypted title failed. Falling back to default title."`) and identifies server type.
 * Device registration — `com/epic/pushnotifications/registration/RegisterDevice`
 (Interconnect command): Request `{ deviceToken, environmentID, supportedDeliveryMethods }` ->
 Response `{ deviceTopic, encryptionKey, eventFrameworkSessionToken, soundConfigurationKey, status }`.
 (deviceToken = FCM token — inferred; the explicit caller was not traced.) `DissociateDevice` Request `{ deviceToken, dissociationSource }` on logout.
 * Payload encryption: `Aes256GcmEncryptionResult`, `EncryptedPayload` (AES-256-GCM with server-provided
 `encryptionKey`; delivery tracking `DeliveredNotification`, commands
 `Epic.Mobile.PushNotifications.ConfirmNotificationDelivery / DismissNotifications /
 GetNotificationPayload / LoadNotification(s) / LoadUserPreferences / GetCustomSoundConfiguration`).
 * VoIP/answering: Twilio Voice FCM (`TwilioFcmProvider`, `VoIP_FCM_REGISTRATION_TIMEOUT_MS`,
 `IncomingCallService`) + `epicFcmProvider/epicFcmToken` vs `vendorFcmProvider/vendorFcmToken`
 split between Epic's own sender and a vendor (Twilio).
* **SignalR fallback** for devices without Google Play services:
 `com.epic.pushnotifications.SignalRPushNotificationService` (a foregroundless Service,
 uses the same `md0/c` client with persisted credentials (preferences keys `EFSessionToken`,
 `EFServerURL`, `EFTopic`), subscribes to the notification topic; server push events arrive as
 `Event.Data` map with keys `title, message, notificationID, UUID, environmentGuid, sound,
 inAppSoundsEnabled, inAppVibrationEnabled, hideIDs, serviceArea, soundVolume, bypassSettings,
 color, icon, customSound`; session expiry triggers a "sign in" notification (`EFLogin`
 PendingIntent -> `ForegroundingActivity`), `KeepSessionAlive` errors close the connection.
* Epic-internal flag `HYPERSPACE_WEB_MIGRATION` ties the newer web-rendered UI to these channels
 (strings); resource `hyperspace_webview`, `hyperspace_host`.

### 3.4 Reimplementation notes for the realtime channel

1. `GET/POST <base>/notificationservices/signalr` (ENv1) — SignalR v2.x-style negotiate/connect;
 querystring `ClientID=<id>`; hub `EventServiceHub` invoke `SubscribeTopic`, receive
 `newSubscribedEvent` / `onEventReceived` invocations; send `SetConnectionParameters`
 `{"session-token": {"token":...}}` after connect (or `Authorization: Bearer` header when OAuth).
2. ENWS: `<base>/eventnotificationservice`, pure WebSocket, hub `EventNotificationHub`,
 `DynamicSubscribe/DynamicUnsubscribe/DynamicPublish`, JSON bodies via the same
 `DynamicSubscribeRequest(DynamicEventConstraints(key, subkey), int, bool, cursor)` shape.
3. Exact SignalR wire format is standard ASP.NET SignalR (pre-3.x) protocol as implemented by
 the bundled Java client (`microsoft/aspnet/signalr/client/transport/WebsocketTransport`,
 `AutomaticTransport`, `hubs/comm/*`) — an OSS-compatible client (e.g. signalr-client-java)
 plus the Epic `framework` shim above should interoperate. Uncertain: whether Epic servers
 require their messagepack/keepALIVE customizations (the fork adds `framework/ENWS*` classes only,
 core client appears stock).


---

## Resolved uncertainties (verification pass 2)

### Perimeter cookie (exact call site, name, scope)

1. **Call site**: `b20.d0.j0(b20.q, continuation)` (private final, invoked from the HSWeb init
 coroutine `b20.d0$q` path — after credentials are obtained (`b20.d0.b0`) and just before
 `webView.loadUrl`):
 ```java
 if (!kotlin.text.s.isBlank(q.c)) { // only when a perimeter token was returned
 internalWebView.getCookieManager // b20.t wrapper -> android.webkit.CookieManager
.setCookie(q.b, // hsWebUrl = <HSUrl>/AutoLogin.aspx (see below)
 q.c, // the perimeter cookie string, VERBATIM
 new b20.d0$s(continuation)); // async setCookie callback
 }
 ```
 (`b20.q` = data class `q(hsWebUrl, authValue, perimeterTokenCookie, loginTicket)`; built in
 `b20.d0.b0` as `new b20.q(P(getHSUrl), LoginToken+"|"+installationID+"|"+codeVerifier,
 resp.getPerimeterAuthenticationToken, resp.getTicket)`.)

2. **Cookie name/domain/path**: the client does **not** construct a cookie string —
 `ObtainEmbeddedHyperspaceCredentials$Response.PerimeterAuthenticationToken` is a complete
 `Set-Cookie`-style string supplied by the Haiku server and passed verbatim to
 `CookieManager.setCookie(url, value)`. Therefore **domain/path scope = the AutoLogin.aspx URL's
 scope** (`CookieManager` default: host-only domain of the HSUrl, path "/" since token has no
 path attr). **Exact cookie NAME is server-controlled and not determinable from the client
 alone** — remains the one open item (a perimeter-protected deployment returns e.g.
 `<PerimeterCookieName>=<token>`; the field name in the JSON schema is
 `PerimeterAuthenticationToken`).

3. **URL derivation** (`b20.d0.P(hsUrl)`): `Uri.parse(hsUrl).buildUpon.appendPath("AutoLogin.aspx")`,
 with Epic-internal test overrides via SharedPreferences `prefListEHSUrlConfig`:
 `elsa` → `https://hsw-fs-test.epic.com/ElsaSDB`, `perimeter` → `https://prj271932.epic.com<same
 path>`, `custom` → `prefEHSCustomUrl`, `environment` (or build flag) → use HSUrl as-is.

4. **Full request decoration** for the AutoLogin.aspx load (same init coroutine):
 - `webView.loadUrl(hsWebUrl, b20.q.a)` where `b20.q.a` returns exactly:
 - `X-Login-Token`: `LoginToken | installationID | codeVerifier` ("|"-joined; PKCE codeVerifier
 from `com.epic.oauth2.n`)
 - `X-Initialization-Ticket`: `Ticket`
 - plus (conditional, when `PerimeterAuthenticationToken` non-blank) the perimeter cookie set above;
 - everything else (session cookies from AutoLogin.aspx response) is managed by `CookieManager`
 thereafter (accept/instance cookies ride the WebView profile; `MULTI_PROFILE` handled by
 `HyperspaceAppWebView.setWebProfile`, `f80.b` WebView subclass confirms shell
 `loadUrl(String, Map)` passthrough).

5. Cookie lifecycle helpers seen in the same package strings (`removeAllCookies`,
 `removeSessionCookies`, `shouldClearCookies`, `removeAllCookies` on logout) — clearing methods
 exist in the account/launch path (`launchCookies` / `removeSessionCookies` strings) but the
 perimeter-specific clearing call was not isolated further; treat logout cookie clearing as present
 but exact method-chain unverified.
