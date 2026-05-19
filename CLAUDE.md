# Traccar Manager — Module Guide

**Repo:** fork of `traccar/traccar-manager` at `mohin1111/traccar-manager`. Current v6.0.5+71.
**Stack:** Flutter / Dart `^3.7.2`. Apache-2.0.
**Role in Transport OS:** **skip.** This is a 5-file WebView wrapper around `traccar-web`, not a separate admin frontend. Rebuilding in a day is faster than forking. Read [`../CLAUDE.md`](../CLAUDE.md) for the wider strategy.

---

## 1. Top-level layout

```
pubspec.yaml             Flutter manifest, v6.0.5+71
lib/                     5 Dart files
android/                 Standard Flutter scaffold + custom drawables, network_security_config, dark styles
ios/                     Standard Flutter scaffold + entitlements, Info.plist with Face ID + background modes
firebase.json            Firebase project config
analysis_options.yaml    Lint config
.github/                 2 workflows + dependabot + funding
tool/                    Release tooling scripts
LICENSE.txt              Apache-2.0
README.md                User-facing docs
```

**Native customizations:**
- Android `app/src/main/`: custom `ic_stat_notify.xml`, `ic_launcher_foreground.xml`, `network_security_config.xml` (allows cleartext for on-prem HTTP servers), light + dark `styles.xml`, boilerplate `MainActivity.kt` (no custom platform channels).
- iOS Runner: `GoogleService-Info.plist`, `Runner.entitlements` (biometric/keychain), `Info.plist` (`NSAllowsArbitraryLoads = true`, `NSFaceIDUsageDescription`, background modes `remote-notification` + `fetch`).

## 2. Build & run

```bash
flutter pub get
flutter run                          # debug
flutter build apk --release          # Android
flutter build ios --release          # iOS (needs signing)
```

- Flutter SDK: `^3.7.2`
- Android: SDK levels delegated to `flutter.minSdkVersion` / `targetSdkVersion` / `compileSdkVersion` in `build.gradle.kts`
- iOS deployment target: **13.0**

## 3. Dependencies

| Package | Purpose |
|---|---|
| `flutter_inappwebview ^6.1.4` | Embedded WebView engine |
| `firebase_core ^4.9.0` | Firebase init |
| `firebase_messaging ^16.2.2` | FCM push |
| `firebase_analytics ^12.4.1` | Usage analytics |
| `firebase_crashlytics ^5.2.2` | Crash reporting |
| `flutter_secure_storage ^10.2.0` | Keychain/Keystore for JWT |
| `local_auth ^3.0.1` | Biometric/Face ID gate on token read |
| `shared_preferences ^2.5.5` | Server URL persistence |
| `shared_preferences_android ^2.4.23` | Android SharedPreferences backend |
| `http ^1.6.0` | Authenticated file downloads |
| `path_provider ^2.1.5` | Temp dir for exports |
| `permission_handler ^12.0.1` | Runtime location + camera permission |
| `share_plus ^13.1.0` | Share exported XLSX/KML/CSV/GPX |
| `url_launcher ^6.3.2` | Open OAuth authorize URLs externally |
| `app_links ^7.0.0` | Deep-link / OAuth redirect handling |
| `rate_my_app ^2.4.0` | App-store rating prompt |

**No paid SDKs.** Firebase: Analytics + Crashlytics + Messaging all configured. `google-services.json` and `GoogleService-Info.plist` present.

## 4. Source tree (`lib/` — 5 files)

| File | Purpose |
|---|---|
| `main.dart` | Entry point; `MainApp` widget; Firebase + Crashlytics init; `RateMyApp` |
| `main_screen.dart` | **Core of the app** — WebView shell, FCM routing, token store, deep links, file download/share |
| `token_store.dart` | `FlutterSecureStorage` + `LocalAuthentication` wrapper for JWT bearer token |
| `error_screen.dart` | Fallback screen when WebView fails; accepts manual URL entry |
| `firebase_options.dart` | Auto-generated `DefaultFirebaseOptions` (per-platform) |

Entry: `main()` → `MainApp` → `MainScreen`. Preferences: `SharedPreferencesWithCache` (key `'url'`).

## 5. Key flows

**WebView URL loading:**
1. `_initWebView()` reads `'url'` from `SharedPreferences` (default `https://demo.traccar.org`).
2. If FCM cold-start carries `eventId`, append `/event/<id>`.
3. `_controller.loadUrl(URLRequest(url: WebUri(...)))`.
4. URL update via JS message `window.appInterface.postMessage('server|<url>')`.

**FCM push routing:**
- `getInitialMessage()` — cold start → route to `/event/<eventId>`
- `onMessageOpenedApp` — background tap → same routing
- `onMessage` — foreground → calls `handleNativeNotification?.(...)` JS bridge + shows `SnackBar`
- Token refresh: calls `updateNotificationToken?.(token)` JS bridge

**Biometric unlock flow:**
1. WebView JS fires `postMessage('login|<jwt>')` on login.
2. `_handleWebMessage` saves JWT to `FlutterSecureStorage`.
3. On next app open, JS fires `postMessage('authentication')`.
4. `TokenStore.read(true)` calls `LocalAuthentication.authenticate(...)` (biometric/PIN prompt).
5. On success, `handleLoginToken?.(token)` JS bridge auto-logs in WebView.

**Server URL config:**
- JS `postMessage('server|<url>')` → `_preferences.setString('url', url)` → reload
- `ErrorScreen` accepts manual URL → same pref key
- Deep link scheme: `org.traccar.manager://...` handled by `_initAppLinks()`

## 6. Native config

**Android `AndroidManifest.xml`:**
- Permissions: `INTERNET`, `USE_BIOMETRIC`, `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `CAMERA`
- Deep-link intent filter: scheme `org.traccar.manager`
- FCM notification icon: `@drawable/ic_stat_notify`
- `networkSecurityConfig` allows HTTP

**iOS `Info.plist`:**
- `NSAllowsArbitraryLoads = true`
- `NSCameraUsageDescription`, `NSFaceIDUsageDescription`, `NSLocationWhenInUseUsageDescription`
- Background modes: `remote-notification`, `fetch`
- URL scheme: `org.traccar.manager`

## 7. CI / release

| Workflow | Purpose |
|---|---|
| `.github/workflows/analyze.yml` | `flutter analyze` lint check on push/PR |
| `.github/workflows/release.yml` | On `v*` tag or manual dispatch: APK build (sign, emulator smoke-test, Google Play production upload) + IPA build (Codemagic signing, archive, App Store Connect upload) |

## 8. What's reusable for Transport OS

**Bluntly: rebuilding the WebView shell is faster than forking.** The entire app is a 5-file wrapper. Things worth examining (not forking):

1. **`token_store.dart` (90 lines)** — clean biometric-gated secure storage. Copy-paste ready.
2. **JS bridge message protocol** — documents the contract between this shell and `traccar-web`:
   - `login|<jwt>` — web sends JWT after successful login
   - `server|<url>` — web changes the configured server URL
   - `authentication` — web requests biometric unlock
   - `authenticated` — flutter signals biometric success
   - `logout` — clear stored token
   - `download|<base64>` — web sends file payload for native download/share
3. **`shouldOverrideUrlLoading` logic** — OAuth redirect interception + external URL open + download intercept. Reusable boilerplate.
4. **`release.yml`** — well-structured dual-platform pipeline worth copying as a template for our greenfield Driver App.

## 9. Things future-Claude must know

This is a 5-file Flutter app whose entire UI is a `flutter_inappwebview` WebView pointing at a Traccar web server URL. The only native logic is (a) the JS bridge message protocol documented above and (b) biometric-gated JWT storage via `token_store.dart`. FCM delivers `eventId` to route the WebView to `/event/<id>`. There is no local UI beyond an error fallback with a URL input box. Server URL persists in `SharedPreferences` under key `'url'`; default `https://demo.traccar.org`.

**If you need a native admin app for Transport OS:** greenfield a thin Flutter shell with `flutter_inappwebview` (or equivalent), copy `token_store.dart` verbatim, and design a fresh JS bridge protocol that fits your web frontend's lifecycle.

**License notice:** Apache-2.0.
