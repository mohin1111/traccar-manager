# token_store.dart

**Role:** Biometric-gated JWT storage. Wraps `flutter_secure_storage` (keychain/keystore) with an optional `local_auth` biometric prompt on read.
**Fits in:** Instantiated once by `_MainScreenState._loginTokenStore`. Used in the `login`/`authentication`/`logout`/`server` JS bridge handlers.
**Read next:** [[main_screen.dart]] (`_handleWebMessage`)

## Public API
- `TokenStore` (lines 7-43) — no constructor args; holds a `FlutterSecureStorage` and a `LocalAuthentication`.
- `save(String token)` (lines 12-19) — store the JWT.
- `read(bool authenticate)` (lines 21-38) — retrieve the JWT, optionally behind a biometric prompt.
- `delete()` (lines 40-42) — remove the JWT.

## Key flows

### `save` (lines 12-19)
Deletes any existing token first, then writes the new one. The delete-before-write avoids a known `flutter_secure_storage` issue on some Android versions where overwriting an existing key can fail silently. Wrapped in try/catch for `PlatformException` (keystore can be unavailable, e.g., right after boot before user unlock).

### `read(authenticate)` (lines 21-38)
1. Return null immediately if no token key exists.
2. If `authenticate == true`: call `_auth.authenticate(localizedReason: ...)` — shows the OS biometric/PIN prompt. If `authenticate == false`, skip the prompt.
3. If authenticated (or prompt skipped): return the stored token.
4. On `LocalAuthException` or `PlatformException`: log and return null.

Called with `authenticate: true` from the `authentication` bridge handler (gate the auto-login), and `authenticate: false` from `_downloadFile` (the user is already in an active session — no need to re-prompt for a file download).

### `delete` (lines 40-42)
Plain delete. Not awaited internally (fire-and-forget) but callers `await` it.

## Gotchas / non-obvious

- **`authenticate` parameter controls the biometric prompt**, not whether to read. `read(false)` still returns the token — it just skips the prompt. Name is slightly misleading; think of it as `promptBiometric`.
- **Delete-before-write in `save`** (line 14) — intentional workaround, not redundant. Don't "optimize" it away.
- **Returns null on every failure path** — caller (`_handleWebMessage`) treats null as "no token, user must log in fresh." Failures (keystore locked, biometric cancelled) are indistinguishable from "no token" by design — safe default.
- **This is the one file genuinely worth lifting** for a Transport OS native app — see parent [`CLAUDE.md`](../../CLAUDE.md). It's ~40 lines, self-contained, copy-paste ready (only deps: `flutter_secure_storage`, `local_auth`, `flutter/services`).

## Line index

- 8 — `_tokenKey = 'token'`
- 12-19 — `save` (note delete-before-write)
- 21-38 — `read` with optional biometric gate
- 26-28 — the `local_auth` prompt
- 40-42 — `delete`
