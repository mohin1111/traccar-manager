# error_screen.dart

**Role:** Fallback screen shown when the WebView fails to load. Displays the error and a text field to manually correct the server URL.
**Fits in:** Rendered by `MainScreen.build` when `_loadingError != null` (instead of the `InAppWebView`).
**Read next:** [[main_screen.dart]] (`build`, lines 248-263)

## Public API
- `ErrorScreen` (lines 4-18) — `StatefulWidget` with three required params:
  - `error` (String) — the error message to display.
  - `url` (String) — pre-fill value for the URL field.
  - `onUrlSubmitted` (`ValueChanged<String>`) — callback invoked with a validated URL.

## Key flows

### Submit (`_submit`, lines 23-33)
1. Trim the text field.
2. Validate: non-empty, parseable as an absolute URI, scheme `http` or `https`.
3. If valid → call `widget.onUrlSubmitted(text)`.
4. If invalid → show "Invalid URL" SnackBar via the global `messengerKey`.

The `onUrlSubmitted` callback (defined in `main_screen.dart:252-261`) deletes the stored token, persists the new URL, and resets WebView state to retry the load.

### Build (lines 47-84)
A centered column: `cloud_off` icon, the error text, and a `TextField` with a check-icon suffix. Both the suffix tap and the keyboard "go" action trigger `_submit`.

## Gotchas / non-obvious

- **Validation lives here, not in the caller** — `main_screen.dart`'s `onUrlSubmitted` trusts the input. So this screen is the URL-validation gate for the error-recovery path. (The `server|` JS bridge message path is separately trusted — web app is assumed well-behaved.)
- **Uses the global `messengerKey`** (line 31) from [[main.dart]] rather than `ScaffoldMessenger.of(context)` — works regardless of widget-tree position.
- **`TextEditingController` created in `initState`, disposed in `dispose`** — standard; pre-filled with the current (failed) URL so the user edits rather than retypes.

## Line index

- 5-7 — the three constructor params
- 23-33 — `_submit` + validation
- 26-27 — the URL validation predicate
- 47-84 — build (icon + error text + URL field)
