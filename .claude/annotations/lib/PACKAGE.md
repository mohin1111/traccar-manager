# `lib/` — Traccar Manager app source

Flutter **WebView shell** that wraps the `traccar-web` React frontend in a native container, adding: FCM push notifications, biometric-gated JWT storage, deep-link / OAuth routing, and native file download/share. It is NOT a separate UI — all screens come from the remote web app.

## File index

| File | One-liner | Annotation |
|---|---|---|
| `main.dart` | App entry; Firebase init; `MainApp` with blue theme + status-bar styling | [main.dart.md](main.dart.md) |
| `main_screen.dart` | The whole app: `InAppWebView` + JS bridge + push routing + OAuth redirect rewriting + file download | [main_screen.dart.md](main_screen.dart.md) |
| `token_store.dart` | Biometric-gated JWT storage in `flutter_secure_storage` | [token_store.dart.md](token_store.dart.md) |
| `error_screen.dart` | Fallback screen on WebView load failure; manual server-URL entry | [error_screen.dart.md](error_screen.dart.md) |
| `firebase_options.dart` | Auto-generated FlutterFire config; project `traccar-manager-app` | [firebase_options.dart.md](firebase_options.dart.md) |

Only 5 files — `main_screen.dart` (~420 lines) is ~80% of the code.

## Dependency graph (intra-package)

```
main.dart
 └── MainApp build → MainScreen (main_screen.dart)
      ├── TokenStore        (token_store.dart) — biometric JWT storage
      └── ErrorScreen       (error_screen.dart) — shown on load failure
firebase_options.dart — referenced for type-safe Firebase config (init uses native config files)
```

`main_screen.dart` is the hub; the other three are leaf dependencies.

## The JS bridge — contract with traccar-web

The web app talks to the native shell via `window.appInterface.postMessage('<verb>|<payload>')`. The shell injects `window.appInterface` as a user script (`main_screen.dart:291-327`) and handles messages in `_handleWebMessage` (lines 193-229):

| Message | Payload | Effect |
|---|---|---|
| `login\|<jwt>` | JWT token | Save token to `TokenStore`; push FCM token to web via `updateNotificationToken?.()` |
| `authentication` | — | Read token from `TokenStore` (biometric prompt); send to web via `handleLoginToken?.()` |
| `authenticated` | — | Completes the `_authenticated` completer (unblocks notification setup) |
| `logout` | — | Delete token from `TokenStore` |
| `download\|<base64>` | base64 xlsx | Decode + save + share as `report.xlsx` |
| `server\|<url>` | server URL | Delete token, persist new URL, reload WebView |

The shell calls INTO the web app via `_controller.evaluateJavascript`:
- `updateNotificationToken?.(token)` — give web the FCM token
- `handleLoginToken?.(token)` — give web a stored JWT for auto-login
- `handleNativeNotification?.(message)` — deliver a foreground push to web

The `?.` optional-call syntax means the web side can omit these handlers safely.

## Major flows (multi-file)

### Cold boot
1. `main()` — Firebase init, Crashlytics error hook, `runApp(MainApp())`.
2. `MainScreen.initState` kicks off three async inits in parallel: `_initWebView`, `_initAppLinks`, `_initNotifications`.
3. `_initWebView` loads SharedPreferences, resolves the server URL (with possible `/event/<id>` suffix from a cold-start push), sets `_settingsReady`.
4. WebView is created → `_controllerReady`. When both ready, `_initialized` completer fires.
5. `_initAppLinks` and `_initNotifications` were awaiting `_initialized` — now they proceed.

### Login + token persistence
1. User logs in on the web app; web fires `login|<jwt>`.
2. `_handleWebMessage` saves JWT via `TokenStore.save`.
3. On next app open, web fires `authentication`; shell reads token via `TokenStore.read(true)` (biometric prompt) and injects it back with `handleLoginToken?.()`.
4. Web confirms login by firing `authenticated` → unblocks notification setup.

### OAuth / OpenID flow (the trickiest part)
1. Web navigates to an authorize URL (detected by `response_type`+`client_id`+`redirect_uri`+`scope` query params present — `main_screen.dart:350`).
2. `_launchAuthorizeRequest` rewrites the `redirect_uri` to a custom-scheme deep link `org.traccar.manager://...` and opens it in the external browser.
3. After auth, the browser redirects to `org.traccar.manager://...`; `_initAppLinks` catches it, rewrites it back to an `https://` URL on the configured server, and loads it in the WebView.

### Push notification
- Cold start: `_initWebView` reads `getInitialMessage()`, appends `/event/<eventId>` to the URL.
- Background tap: `onMessageOpenedApp` loads `/event/<eventId>`.
- Foreground: `onMessage` injects `handleNativeNotification?.()` into the web app + shows a native SnackBar.

### File download
- xlsx Blob created in-page → intercepted by patched `URL.createObjectURL` (injected script) → `download|<base64>` message → native save + share.
- Direct file URLs (xlsx/kml/csv/gpx) → `shouldOverrideUrlLoading` cancels navigation, `_downloadFile` fetches with `Bearer` token, saves + shares.

## Conventions

- **Two `Completer`s coordinate async init**: `_initialized` (settings + controller ready) and `_authenticated` (web confirmed login). Other inits await these instead of using flags + polling.
- **Server URL persists under SharedPreferences key `'url'`** (default `https://demo.traccar.org`).
- **All web↔native comms go through the JS bridge** documented above. No other channel.

## License notice

- App source: Apache-2.0.
- No paid SDKs (unlike [[client]] which uses Transistorsoft).

## Recommendation for Transport OS

Per the parent [`CLAUDE.md`](../../CLAUDE.md): this module is **not worth forking** — it's a thin WebView wrapper. If a native admin app is ever needed, rebuild the shell and copy `token_store.dart` verbatim. The valuable artifact here is the **JS bridge protocol** (documented above) — reuse the pattern, design your own verbs.

## How to add a new JS bridge message

1. Web side: call `window.appInterface.postMessage('newverb|payload')`.
2. Native side: add a `case 'newverb':` in `_handleWebMessage` (`main_screen.dart:195-228`).
3. To call back into web: `_controller?.evaluateJavascript(source: "webHandler?.(...)")`.
