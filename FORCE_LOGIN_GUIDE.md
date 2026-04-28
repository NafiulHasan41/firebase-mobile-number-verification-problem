# Force Login (Device Switch) — Implementation Guide

When a user tries to log in on a new device while already active on another device,
the backend returns `{"status": "device_conflict"}`. The user must explicitly confirm
they want to switch devices. This guide covers how to handle that for each auth method.

---

## Google / Apple / Email+Password

These three providers go through Firebase SDK first, so the app already has a
`firebase_id_token` by the time the conflict is detected.

### Flow Overview

```
                     NEW DEVICE (B)                          OLD DEVICE (A)
                     ─────────────                          ──────────────
1. Firebase SDK login → idToken_v1                         (has active session)
2. POST /auth/login(idToken_v1, deviceB) → device_conflict
3. Show "Switch device?" dialog
4. User taps "Switch Device"
5. POST /auth/force-login(idToken_v1, deviceB) → ok
                                  │
                                  └──► Backend revokes ALL Firebase
                                       refresh tokens for this user.
                                       Sets revoked_before timestamp.
                                       Sets activeDevice = B.
6. ⚠ idToken_v1 is now REVOKED.
   Call getIdToken(forceRefresh: true)
   to get idToken_v2.
7. Use idToken_v2 for all future API calls
                                                          8. Next API call:
                                                             401 (token revoked)
                                                          9. App tries refresh:
                                                             getIdToken(true)
                                                             → throws (refresh
                                                                token revoked)
                                                          10. Show login screen.
```

### Two things to handle

**A) On Device B (the one switching in):** the token you used to call `/auth/force-login` was just invalidated by the backend. Before any other API call, you must call `getIdToken(forceRefresh: true)` to get a fresh ID token. Otherwise your next request returns 401.

**B) On Device A (the one being kicked out):** any API call after the force-login returns 401, and `getIdToken(forceRefresh: true)` will throw because the underlying Firebase refresh token has been revoked. Catch this in your global 401 interceptor, sign the user out locally, and route to the login screen.

### Endpoint

```
POST /api/v1/auth/force-login
```

**Body:**
```json
{
  "firebase_id_token": "eyJhbGci...",
  "device_id": "device-xyz"
}
```

**Response (200):**
```json
{
  "status": "ok",
  "message": "Other devices signed out.",
  "user_id": "550e8400-..."
}
```

### Flutter Example — Full Flow

```dart
Future<bool> handleLogin(String firebaseIdToken, String deviceId) async {
  // 1. Try normal login first
  final resp = await api.post('/auth/login', body: {
    'firebase_id_token': firebaseIdToken,
    'device_id': deviceId,
  });

  if (resp['status'] == 'ok') {
    return true; // No conflict — proceed to app
  }

  if (resp['status'] != 'device_conflict') {
    return false;
  }

  // 2. Conflict — ask the user
  final confirmed = await showSwitchDeviceDialog();
  if (!confirmed) {
    // Sign out of Firebase locally so we don't leave a half-state behind
    await FirebaseAuth.instance.signOut();
    return false;
  }

  // 3. Force-login (uses the same idToken — still valid at this moment)
  final force = await api.post('/auth/force-login', body: {
    'firebase_id_token': firebaseIdToken,
    'device_id': deviceId,
  });
  if (force['status'] != 'ok') return false;

  // 4. ⚠ CRITICAL: the idToken we just used has been revoked by the backend.
  //    We must refresh BEFORE making any other API call.
  final freshIdToken = await FirebaseAuth.instance
      .currentUser
      ?.getIdToken(true); // forceRefresh = true

  if (freshIdToken == null) {
    // Firebase refresh failed — fall back to manual login
    await FirebaseAuth.instance.signOut();
    return false;
  }

  // 5. Use freshIdToken for all subsequent requests.
  //    Update whatever holds your auth header (interceptor, ApiClient, etc.)
  api.setIdToken(freshIdToken);

  return true;
}
```

### Flutter — 401 interceptor (handles getting kicked on Device A)

Every API call should go through an interceptor that handles 401 by trying to refresh once. If refresh itself fails, the user has been kicked — log them out locally.

```dart
Future<http.Response> authedRequest(
  String method, String path, {Map? body}
) async {
  var resp = await _send(method, path, body: body);

  if (resp.statusCode != 401) return resp;

  // Try to refresh
  String? freshToken;
  try {
    freshToken = await FirebaseAuth.instance
        .currentUser
        ?.getIdToken(true);
  } catch (_) {
    freshToken = null;
  }

  if (freshToken == null) {
    // Refresh token revoked → user was kicked from another device
    await FirebaseAuth.instance.signOut();
    await secureStorage.deleteAll(); // clear biometric_secret etc.
    navigateToLoginScreen();
    return resp;
  }

  api.setIdToken(freshToken);
  return _send(method, path, body: body); // retry once
}
```

### Swift Example — Full Flow

```swift
func handleLogin(firebaseIdToken: String, deviceId: String) async throws -> Bool {
    // 1. Try normal login first
    let resp = try await api.post("/auth/login", body: [
        "firebase_id_token": firebaseIdToken,
        "device_id": deviceId,
    ])

    if resp["status"] as? String == "ok" { return true }
    guard resp["status"] as? String == "device_conflict" else { return false }

    // 2. Conflict — ask the user
    let confirmed = await showSwitchDeviceDialog()
    guard confirmed else {
        try? Auth.auth().signOut()
        return false
    }

    // 3. Force-login
    let force = try await api.post("/auth/force-login", body: [
        "firebase_id_token": firebaseIdToken,
        "device_id": deviceId,
    ])
    guard force["status"] as? String == "ok" else { return false }

    // 4. ⚠ CRITICAL: refresh the idToken before any other API call.
    guard let freshIdToken = try? await Auth.auth()
        .currentUser?
        .getIDTokenForcingRefresh(true)
    else {
        try? Auth.auth().signOut()
        return false
    }

    // 5. Use freshIdToken for everything from here.
    api.setIdToken(freshIdToken)
    return true
}
```

### Swift — 401 interceptor

```swift
func authedRequest(_ request: URLRequest) async throws -> Data {
    var (data, resp) = try await URLSession.shared.data(for: request)
    guard (resp as? HTTPURLResponse)?.statusCode == 401 else { return data }

    // Try to refresh
    let fresh: String?
    do {
        fresh = try await Auth.auth()
            .currentUser?
            .getIDTokenForcingRefresh(true)
    } catch {
        fresh = nil
    }

    guard let freshToken = fresh else {
        // Refresh token revoked → user was kicked
        try? Auth.auth().signOut()
        try? KeychainHelper.deleteAll()
        await MainActor.run { navigateToLoginScreen() }
        return data
    }

    api.setIdToken(freshToken)
    var retry = request
    retry.setValue("Bearer \(freshToken)", forHTTPHeaderField: "Authorization")
    (data, _) = try await URLSession.shared.data(for: retry)
    return data
}
```

### Why the Refresh Is Required

When you call `/auth/force-login`:

1. The backend calls Firebase Admin's `revoke_refresh_tokens(uid)` — Firebase records `tokensValidAfterTime = now`.
2. The backend also stores `revoked_before:{uid} = now` in Redis (covers the ~1 hour window where existing ID tokens haven't expired yet).
3. **Every existing ID token for this user is now invalid**, including the one you just sent.
4. The backend's auth dependency (`deps.py`) checks `token_iat < revoked_before` on every request and returns 401 if true.

`getIdToken(forceRefresh: true)` mints a brand-new ID token whose `iat > revoked_before` → passes the check.

> Note: phone+password users do NOT need this manual refresh step. `/auth/phone-login(force=true)` returns a fresh `custom_token` — calling `signInWithCustomToken` with it produces a brand-new Firebase session whose tokens are issued AFTER the revocation. See the Phone+Password section below.

---

## Phone + Password

Phone users authenticate against the backend directly (no Firebase SDK step before
the conflict check). When `device_conflict` is returned, the app has no
`firebase_id_token` — so `/auth/force-login` cannot be used.

Instead, `/auth/phone-login` accepts a `force: true` flag that handles the device
switch internally and returns a fresh `custom_token`.

### Flow

```
1. POST /auth/phone-login(phone, password, device_id) → device_conflict
2. Show dialog: "You're signed in on another device. Switch here?"
3. User taps "Switch Device"
4. POST /auth/phone-login(phone, password, device_id, force: true) → {custom_token}
5. FirebaseAuth.signInWithCustomToken(custom_token) → Firebase session
6. POST /auth/login(firebase_id_token, device_id) → {"status": "ok"}
7. Proceed to app normally
```

### Endpoint

```
POST /api/v1/auth/phone-login
```

**Body (normal login):**
```json
{
  "phone": "+1234567890",
  "password": "MyPass123",
  "device_id": "device-xyz"
}
```

**Body (force switch):**
```json
{
  "phone": "+1234567890",
  "password": "MyPass123",
  "device_id": "device-xyz",
  "force": true
}
```

**Response on conflict (force=false or omitted):**
```json
{
  "status": "device_conflict",
  "message": "Active session on another device. Use force=true to switch."
}
```

**Response on force=true (200):**
```json
{
  "status": "ok",
  "custom_token": "eyJhbGci...",
  "user_id": "550e8400-..."
}
```

### Flutter Example

```dart
Future<void> handlePhoneLogin(String phone, String password, String deviceId) async {
  // Step 1 — normal login attempt
  final resp = await api.post('/auth/phone-login', body: {
    'phone': phone,
    'password': password,
    'device_id': deviceId,
  });

  if (resp['status'] == 'ok') {
    // No conflict — sign in with custom token
    await _signInWithCustomToken(resp['custom_token'], deviceId);
    return;
  }

  if (resp['status'] == 'device_conflict') {
    // Step 2 — show dialog
    final confirmed = await showSwitchDeviceDialog();
    if (!confirmed) return;

    // Step 3 — force login with same credentials
    final forceResp = await api.post('/auth/phone-login', body: {
      'phone': phone,
      'password': password,
      'device_id': deviceId,
      'force': true,
    });

    await _signInWithCustomToken(forceResp['custom_token'], deviceId);
  }
}

Future<void> _signInWithCustomToken(String customToken, String deviceId) async {
  // Exchange custom token for Firebase ID token
  final credential = await FirebaseAuth.instance.signInWithCustomToken(customToken);
  final idToken = await credential.user?.getIdToken();

  // Register with backend
  await api.post('/auth/login', body: {
    'firebase_id_token': idToken,
    'device_id': deviceId,
  });
}
```

### Swift Example

```swift
func handlePhoneLogin(phone: String, password: String, deviceId: String) async throws {
    // Step 1 — normal login attempt
    let resp = try await api.post("/auth/phone-login", body: [
        "phone": phone,
        "password": password,
        "device_id": deviceId,
    ])

    if resp["status"] as? String == "ok" {
        try await signInWithCustomToken(resp["custom_token"] as! String, deviceId: deviceId)
        return
    }

    if resp["status"] as? String == "device_conflict" {
        // Step 2 — show dialog
        let confirmed = await showSwitchDeviceDialog()
        guard confirmed else { return }

        // Step 3 — force login with same credentials
        let forceResp = try await api.post("/auth/phone-login", body: [
            "phone": phone,
            "password": password,
            "device_id": deviceId,
            "force": true,
        ])

        try await signInWithCustomToken(forceResp["custom_token"] as! String, deviceId: deviceId)
    }
}

func signInWithCustomToken(_ customToken: String, deviceId: String) async throws {
    // Exchange custom token for Firebase ID token
    let result = try await Auth.auth().signIn(withCustomToken: customToken)
    let idToken = try await result.user.getIDToken()

    // Register with backend
    try await api.post("/auth/login", body: [
        "firebase_id_token": idToken,
        "device_id": deviceId,
    ])
}
```

---

## Summary

| Auth Method | Conflict Endpoint | Force Mechanism |
|---|---|---|
| Google | `POST /auth/login` → conflict → `POST /auth/force-login` | Same ID token from step 1 |
| Apple | `POST /auth/login` → conflict → `POST /auth/force-login` | Same ID token from step 1 |
| Email + Password | `POST /auth/login` → conflict → `POST /auth/force-login` | Same ID token from step 1 |
| Phone + Password | `POST /auth/phone-login` → conflict → `POST /auth/phone-login(force=true)` | `force: true` flag, no separate endpoint |

**Key difference:** Phone users never have a `firebase_id_token` at the time of conflict,
so they use the `force` flag on `/auth/phone-login` instead of `/auth/force-login`.
`/auth/force-login` is only for Google, Apple, and Email users.
