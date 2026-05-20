# firebase_options.dart

**Role:** Auto-generated FlutterFire config. Holds API keys, app IDs, bundle IDs for the upstream Traccar Manager Firebase project (`traccar-manager-app`).
**Fits in:** Available for `Firebase.initializeApp(options: ...)` — but `main.dart:10` calls `Firebase.initializeApp()` without options, so native config files (`google-services.json` / `GoogleService-Info.plist`) are used instead. This file is vestigial in the current codebase.
**Read next:** [[main.dart]]

## Public API
- `DefaultFirebaseOptions.currentPlatform` (lines 18-50) — platform-dispatch getter; throws `UnsupportedError` for web/macOS/Windows/Linux.
- `DefaultFirebaseOptions.android` (lines 52-59) — Firebase project `traccar-manager-app`.
- `DefaultFirebaseOptions.ios` (lines 61-70) — iOS bundle ID `org.traccar.TraccarManager`.

## Gotchas / non-obvious

- **Do NOT edit by hand.** Regenerate via `flutterfire configure` against your own Firebase project.
- **Currently unused by `main.dart`** — `Firebase.initializeApp()` is called without `options`, so native config files are consulted. This file exists for explicit/type-safe init only.
- **Firebase client API keys are public by design** — auth is via project rules, not key secrecy. But they DO bind the app to the `traccar-manager-app` Firebase project.
- **iOS bundle ID is `org.traccar.TraccarManager`** (line 69) — note this differs from the `org.traccar.manager` *deep-link scheme* used for OAuth in [[main_screen.dart]]. The scheme and the bundle ID are independent identifiers; don't conflate them.
- **For a Transport OS native admin app:** create a new Firebase project, run `flutterfire configure`, rebuild — replaces this whole file.

## Line index

- 18-50 — platform dispatch
- 52-59 — Android config (project `traccar-manager-app`)
- 61-70 — iOS config
- 69 — iOS bundle ID `org.traccar.TraccarManager`
