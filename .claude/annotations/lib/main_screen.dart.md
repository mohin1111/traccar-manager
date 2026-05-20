# main_screen.dart

**Role:** The entire app. An `InAppWebView` wrapping the remote `traccar-web` frontend, plus: JS bridge, FCM push routing, OAuth/OpenID redirect rewriting, deep-link handling, native file download/share, error fallback, hardware back-button handling.
**Fits in:** Mounted by `MainApp.build`. ~420 lines — ~80% of the module.
**Read next:** [[token_store.dart]], [[error_screen.dart]], [[PACKAGE]] (JS bridge contract table)

## Public API
- `MainScreen` (lines 23-28) — `StatefulWidget`, no params.
- `_MainScreenState` (lines 30-419) — holds the WebView controller, preferences, completers, token store.

## State fields (lines 31-45)
- `_urlKey = 'url'` — SharedPreferences key for the server URL.
- `_initialized` — `Completer<void>` fired when settings AND controller are both ready.
- `_authenticated` — `Completer<void>` fired when web confirms login (`authenticated` message).
- `_preferences`, `_appLinks`, `_appLinksSubscription` — late-init plumbing.
- `_loginTokenStore` — a [[token_store]] `TokenStore` instance.
- `_controller` — the `InAppWebViewController` (null until WebView created).
- `_loadingError`, `_initialUrl`, `_settingsReady`, `_controllerReady` — render-state flags.

## Key flows

### Init (`initState` → 3 parallel inits, lines 47-53)
`_initWebView`, `_initAppLinks`, `_initNotifications` all kicked off together. The latter two `await _initialized.future` so they effectively run after `_initWebView` + WebView creation complete.

### `_initWebView` (lines 146-169)
1. Create `SharedPreferencesWithCache` (allowList `{'url'}`, legacy Android backend).
2. Resolve server URL via `_getUrl()` (default `https://demo.traccar.org`, trailing slash stripped).
3. Check `getInitialMessage()` — if a cold-start push carries `eventId`, append `/event/<eventId>`.
4. `setState` → `_initialUrl`, `_settingsReady = true`.
5. `_maybeCompleteInitialized()`.

### `_maybeCompleteInitialized` (lines 140-144)
Fires the `_initialized` completer only when **both** `_settingsReady` (from `_initWebView`) and `_controllerReady` (from `onWebViewCreated`) are true. This is the synchronization barrier.

### JS bridge (`_handleWebMessage`, lines 193-229)
Messages arrive as `'<verb>|<payload>'`. Split on `|`, switch on `parts[0]`:
- `login` — save JWT to [[token_store]], push FCM token to web.
- `authentication` — read JWT (biometric prompt), inject via `handleLoginToken?.()`.
- `authenticated` — complete `_authenticated` completer.
- `logout` — delete stored token.
- `download` — base64-decode payload, save + share as `report.xlsx`.
- `server` — delete token, persist new URL, reload WebView.

Full contract table in [[PACKAGE]].

### `_initNotifications` (lines 171-191)
After `_initialized`:
1. `onMessageOpenedApp` — background push tap → load `/event/<eventId>`.
2. Request FCM permission.
3. Wait up to 30s for `_authenticated` (don't push tokens before login).
4. `onTokenRefresh` → inject `updateNotificationToken?.()`.
5. `onMessage` (foreground) → inject `handleNativeNotification?.()` + native SnackBar.

### OAuth redirect rewriting (the trickiest logic)
**Outbound** (`shouldOverrideUrlLoading` → `_launchAuthorizeRequest`, lines 344-353 + 80-96):
- A navigation is detected as an authorize request if its query has all of `response_type`, `client_id`, `redirect_uri`, `scope`.
- `_launchAuthorizeRequest` rewrites the `redirect_uri` to a custom-scheme URI `org.traccar.manager://<host>/<path>` so the OAuth provider redirects back into the app instead of a web page.
- Opens it in the external browser; cancels the in-WebView navigation.

**Inbound** (`_initAppLinks`, lines 55-78):
- App receives `org.traccar.manager://...` deep link.
- Rewrites it back to `https://<configured-server>/<path>` (host+path from the URI become the path on the real server).
- If a `code` param is present, also sets `redirect_uri` so the web app can complete the token exchange.
- Loads the rewritten URL in the WebView.

### File download (two paths)
1. **Blob interception** — injected user script (lines 304-315) patches `URL.createObjectURL`; when an xlsx Blob is created, reads it as base64 and fires `download|<base64>`.
2. **Direct file URL** — `shouldOverrideUrlLoading` detects xlsx/kml/csv/gpx extensions (`_isDownloadable`, lines 109-112), cancels navigation, `_downloadFile` (lines 123-138) fetches with `Bearer` token, then `_shareFile`.

### `_shareFile` (lines 114-121)
Writes bytes to external storage (Android) / documents dir (iOS), then `SharePlus.instance.share`.

### Hardware back button (`PopScope`, lines 264-277)
`canPop: false` intercepts the back gesture. If the WebView can go back AND isn't at root/login, `goBack()`. Otherwise `SystemNavigator.pop()` (exit app).

### Injected user script (lines 291-327)
Injected `AT_DOCUMENT_START`:
- Defines `window.appInterface.postMessage` — queues messages if the Flutter bridge isn't ready yet, flushes the queue on `flutterInAppWebViewPlatformReady`.
- Patches `URL.createObjectURL` for xlsx Blob interception.

## Gotchas / non-obvious

- **Two completers, not flags+polling** — `_initialized` and `_authenticated` cleanly sequence async setup. `_initNotifications` and `_initAppLinks` await `_initialized`; token push awaits `_authenticated` (with a 30s timeout fallback so it doesn't hang forever if the user never logs in).
- **The message queue in the injected script** (lines 299-301, 316-323) handles the race where the web app calls `postMessage` before `flutter_inappwebview` finishes initializing — without it, early messages are lost.
- **OAuth round-trip is two rewrites** — outbound `https → org.traccar.manager` on `redirect_uri`, inbound `org.traccar.manager → https` on the whole URL. Get one wrong and login silently fails. This is the single most fragile code in the module.
- **`key: ValueKey(_initialUrl)` on `InAppWebView`** (line 283) — forces a full WebView rebuild when the server URL changes (e.g., after a `server|` message or error-screen submit). Without it, the old WebView state persists.
- **`_isRootOrLogin`** (lines 231-237) — back button at `/` or `/login` exits the app rather than navigating WebView history into a weird state.
- **`onRenderProcessGone` / `onWebContentProcessDidTerminate`** (lines 380-385) — the OS can kill the WebView renderer under memory pressure; these reload it. Without them the app shows a blank screen.
- **iOS error code 102** (line 370) — "frame load interrupted" is benign on iOS (happens on redirects); explicitly ignored so it doesn't trip the error screen.
- **`download|` always names the file `report.xlsx`** (line 219) — the Blob-intercept path loses the original filename. The direct-URL path (line 131) preserves the extension via timestamp naming.
- **Permission handling** — `onGeolocationPermissionsShowPrompt` (geolocation) and `onPermissionRequest` (camera/video only; other resources denied) bridge web permission requests to native `permission_handler`.

## Line index

- 47-53 — `initState` three parallel inits
- 55-78 — `_initAppLinks` (inbound OAuth deep-link rewrite)
- 80-96 — `_launchAuthorizeRequest` (outbound OAuth redirect rewrite)
- 104-107 — `_getUrl` (server URL resolution)
- 109-112 — `_isDownloadable`
- 114-121 — `_shareFile`
- 123-138 — `_downloadFile`
- 140-144 — `_maybeCompleteInitialized` (sync barrier)
- 146-169 — `_initWebView`
- 171-191 — `_initNotifications`
- 193-229 — `_handleWebMessage` (JS bridge — the core contract)
- 231-237 — `_isRootOrLogin`
- 264-277 — `PopScope` back-button handling
- 291-327 — injected user script (appInterface + Blob patch)
- 344-367 — `shouldOverrideUrlLoading` (OAuth detect, external-host, downloads)
- 380-385 — renderer-death recovery
