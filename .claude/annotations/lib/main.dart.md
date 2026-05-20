# main.dart

**Role:** App entry point. Initializes Firebase, wires Crashlytics, mounts `MainApp` (theming + status-bar styling + `MainScreen`).
**Fits in:** Invoked by Flutter framework on launch. Thin — all real logic is in [[main_screen.dart]].
**Read next:** [[main_screen.dart]]

## Public API
- `main()` (lines 8-13) — async entry; `ensureInitialized` → Firebase → Crashlytics hook → `runApp`.
- `messengerKey` (line 15) — global `GlobalKey<ScaffoldMessengerState>` for showing SnackBars from outside the widget tree (used by [[main_screen.dart]] push handler and [[error_screen.dart]]).
- `MainApp` (lines 17-66) — `StatefulWidget` root; provides `MaterialApp` with blue-seed light/dark themes.

## Key flows

### Boot (`main`, lines 8-13)
1. `WidgetsFlutterBinding.ensureInitialized()`.
2. `Firebase.initializeApp()` — no `options` arg; uses native config files.
3. `FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError` — forward uncaught Flutter errors to Crashlytics.
4. `runApp(MainApp())`.

Note: unlike [[client]]'s `main.dart`, there is **no Preferences/Geolocation/Push init here** — `MainScreen.initState` does all of that, because this app's state is much simpler (just a server URL).

### Rate-my-app (lines 27-36)
`RateMyApp()` with default thresholds (unlike client which uses `minDays: 0, minLaunches: 0`). Prompts after the default usage period.

### Status-bar styling (`builder`, lines 55-63)
The `MaterialApp.builder` callback runs on every rebuild; it sets `SystemUiOverlayStyle` so status-bar icons contrast with the current platform brightness (light icons on dark, dark on light).

## Gotchas / non-obvious

- **Blue seed colour** (`Colors.blue`, lines 44/50) — Manager's visual identity. Contrast with [[client]] which uses green.
- **`messengerKey` is defined here** but used heavily in [[main_screen.dart]] and [[error_screen.dart]] — they import `package:traccar_manager/main.dart` for it.
- **`builder` re-applies overlay style on every frame** — slightly wasteful but ensures correctness across theme changes; standard Flutter pattern.
- **No `options:` passed to `Firebase.initializeApp()`** — relies on `google-services.json` / `GoogleService-Info.plist`. [[firebase_options.dart]] exists but is not wired in.

## Line index

- 8-13 — `main()` boot
- 15 — `messengerKey` global
- 25 — `RateMyApp` instance
- 30-35 — rate dialog post-frame callback
- 40-64 — `MaterialApp` with themes
- 54 — `home: MainScreen()`
- 55-63 — status-bar overlay styling
