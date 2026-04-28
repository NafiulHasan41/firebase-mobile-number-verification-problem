# Biometric Login Bug — Root Cause & Fix

---

## Her Code (the broken part)

### `enableBiometric()` — where the bug is

```dart
// Call backend to enable flag
final token = await refreshIdToken();   // ← gets Firebase ID token
if (token == null) { ... }

final response = await http.post(url, headers: {
  'Authorization': 'Bearer $token',     // ← correct: ID token used for API auth
  ...
});

if (response.statusCode == 200) {
  // Store refresh token securely
  await _secureStorage.write(
    key: _keyRefreshToken,
    value: token,                        // ← BUG: storing the ID token, not the refresh token
  );
}
```

### `refreshIdToken()` — what it actually returns

```dart
Future<String?> refreshIdToken({bool forceRefresh = false}) async {
  final user = _firebaseAuth.currentUser;
  _idToken = await user.getIdToken(forceRefresh);  // ← returns ID token (JWT)
  return _idToken;
}
```

### `loginWithBiometric()` — where the crash happens

```dart
final refreshToken = await _secureStorage.read(key: _keyRefreshToken);
// refreshToken is actually an ID token here — wrong type

final idToken = await _exchangeRefreshTokenForIdToken(refreshToken);
// passes the ID token to Firebase REST API as if it's a refresh token → fails
```

### `_exchangeRefreshTokenForIdToken()` — correct implementation, wrong input

```dart
final response = await http.post(
  Uri.parse('https://securetoken.googleapis.com/v1/token?key=$apiKey'),
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'grant_type=refresh_token&refresh_token=$refreshToken',
  // Firebase receives a JWT here instead of a refresh token → rejects it
);
```

---

## Why It's Wrong

There are **two different Firebase tokens** and they are not interchangeable:

| Token | How to get | Expires | Purpose |
|---|---|---|---|
| **ID token** | `user.getIdToken()` | 60 minutes | Used in `Authorization: Bearer` header for API calls |
| **Refresh token** | `user.refreshToken` | Never (unless revoked) | Used to get new ID tokens via Firebase REST API |

`refreshIdToken()` returns an **ID token** — its name is misleading. Despite the word "refresh" in the method name, it is calling `user.getIdToken()` which returns a short-lived JWT.

The token she logged starts with `eyJhbGciOiJSUzI1NiI...` — that is a JWT (ID token). Firebase refresh tokens look completely different (no dots, no base64 structure).

When `loginWithBiometric()` reads that stored value and calls the Firebase REST API with `grant_type=refresh_token`, Firebase rejects it with `invalid_grant` / `invalid-custom-token` because a JWT is not a valid refresh token.

---

## What Needs to Change

**Only one line in `enableBiometric()` needs to change** — the value being written to secure storage:

```dart
if (response.statusCode == 200) {
  // BEFORE (wrong):
  await _secureStorage.write(
    key: _keyRefreshToken,
    value: token,                                          // ID token
  );

  // AFTER (correct):
  final refreshToken = FirebaseAuth.instance.currentUser?.refreshToken;
  await _secureStorage.write(
    key: _keyRefreshToken,
    value: refreshToken,                                   // actual refresh token
  );
}
```

Keep `token` (from `refreshIdToken()`) for the `Authorization` header — that usage is correct. Just store `currentUser?.refreshToken` separately for biometric login.

**Everything else is correct** — `_exchangeRefreshTokenForIdToken()`, `loginWithBiometric()`, `disableBiometric()`, the REST API call structure — all fine.
