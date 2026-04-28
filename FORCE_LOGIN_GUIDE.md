# Force Login (Device Switch) — Implementation Guide

When a user tries to sign in on a new device while another device is already
active, the backend returns `{"status": "device_conflict"}`. The user must
explicitly confirm they want to switch devices.

This guide covers the **unified custom_token flow** that works for all four
providers (Google, Apple, Email+Password, Phone+Password). After force-login,
the client always receives a `custom_token` that it exchanges via
`signInWithCustomToken` for a fresh Firebase session — no re-authentication
prompts, no per-provider branching.

---

## The Core Idea

After `/auth/force-login` (or `/auth/phone-login` with `force=true`), the
backend has done two things:

1. **Revoked every existing refresh token** for the user — every device,
   including the one that just made the request.
2. **Minted a new `custom_token`** that survives the revocation, because it
   was created *after* the `tokensValidAfterTime` cutoff.

So the client never tries to re-use or refresh the old token. It signs in
with the custom token, gets a brand-new Firebase session, and continues
from there.

---

## Universal Flow Diagram

```
                NEW DEVICE (B)                            OLD DEVICE (A)
                ─────────────                             ──────────────
1. Authenticate via Firebase                              (has active session)
   (Google / Apple / Email / Phone)
   → idToken_v1
2. POST /auth/login(idToken_v1, deviceB)
   → device_conflict
3. Show "Switch device?" dialog
4. User taps "Switch"
5. POST /auth/force-login(idToken_v1, deviceB)
   (or /auth/phone-login force=true for phone)
   → { custom_token, ok }
                                  │
                                  └──► Backend revokes all refresh tokens.
                                       Mints fresh custom_token AFTER revoke.
                                       Sets activeDevice = B.
6. signInWithCustomToken(custom_token)
   → fresh Firebase session
7. currentUser.getIdToken() → idToken_v2
8. POST /auth/login(idToken_v2, deviceB) → ok
9. Continue using app                                    10. Next API call:
                                                            401 (token revoked)
                                                         11. App's 401 interceptor
                                                            tries getIdToken(true).
                                                            Throws (refresh token
                                                            revoked).
                                                         12. App signs out locally,
                                                            shows login screen.
```

---

## Endpoints

### Google / Apple / Email+Password

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
  "user_id": "550e8400-...",
  "custom_token": "eyJhbGci..."
}
```

### Phone + Password

Phone users authenticate against the backend directly — they have no
`firebase_id_token` at the moment of conflict. The same idea is implemented
inline on `/auth/phone-login` via a `force: true` flag.

```
POST /api/v1/auth/phone-login
```

**Body (normal):**
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

**Response (200) on conflict (force=false):**
```json
{
  "status": "device_conflict",
  "message": "Active session on another device. Use force=true to switch."
}
```

**Response (200) on force=true:**
```json
{
  "status": "ok",
  "custom_token": "eyJhbGci...",
  "user_id": "550e8400-..."
}
```

---

## What To Do After You Receive `custom_token`

The `custom_token` returned by `/auth/force-login` (or `/auth/phone-login` with
`force=true`) is **not** a Firebase ID token. You cannot put it in the
`Authorization` header. It is a short-lived authentication credential that
Firebase Auth accepts via one specific method: `signInWithCustomToken`.

Run these four steps **in order**, every time you receive a `custom_token`:

### Step 1 — Sign in to Firebase with the custom token

```dart
final cred = await FirebaseAuth.instance.signInWithCustomToken(custom_token);
```
```swift
let cred = try await Auth.auth().signIn(withCustomToken: customToken)
```

This creates a brand-new Firebase session. The Firebase SDK now has a fresh
refresh token internally — the revocation no longer affects you because this
session was created *after* the cutoff.

### Step 2 — Get a fresh Firebase ID token

```dart
final freshIdToken = await cred.user?.getIdToken();
```
```swift
let freshIdToken = try await cred.user.getIDToken()
```

This is the JWT you'll send in the `Authorization: Bearer <token>` header for
all future API calls.

### Step 3 — Update your API client's stored token

Your app almost certainly has an `ApiClient` / interceptor / Dio singleton
that holds the current ID token. **Replace it with `freshIdToken`.** The token
you used before force-login is dead — every subsequent request must use the
new one.

```dart
api.setIdToken(freshIdToken);
```
```swift
api.setIdToken(freshIdToken)
```

### Step 4 — Register the session with the backend

```
POST /auth/login
{
  "firebase_id_token": <freshIdToken>,
  "device_id": <deviceId>
}
```

This tells the backend "this device now holds the active session." After this
call returns `{"status": "ok"}`, the user is fully signed in and you can
navigate to the home screen.

### Why all four steps are needed

- **Skip Step 1** → no Firebase session at all, every API call fails.
- **Skip Step 2** → you'd try to send the `custom_token` as a Bearer token, but the backend expects an ID token. 401.
- **Skip Step 3** → your interceptor keeps sending the old (revoked) token. 401 on every call.
- **Skip Step 4** → backend doesn't know which device owns the session. The next force-login from elsewhere wouldn't kick you, etc.

### TL;DR

```
custom_token
   │
   ▼
signInWithCustomToken          ← Step 1
   │
   ▼
currentUser.getIdToken()       ← Step 2
   │
   ▼
api.setIdToken(...)            ← Step 3
   │
   ▼
POST /auth/login               ← Step 4
   │
   ▼
home screen
```

The Flutter and Swift sections below wrap exactly these four steps inside a
helper called `_completeWithCustomToken` / `completeWithCustomToken`. You call
it once, after you receive the custom_token from either `/auth/force-login`
or `/auth/phone-login(force=true)`.

---

## Flutter — Universal Handler

Same code path for all four providers. The only difference is how you obtain
the initial `firebase_id_token` (Google/Apple/Email use Firebase SDK first,
phone uses backend).

```dart
import 'package:firebase_auth/firebase_auth.dart';

/// For Google / Apple / Email — caller has already done Firebase SDK login
/// and has a fresh firebase_id_token.
Future<bool> handleLogin(String firebaseIdToken, String deviceId) async {
  // 1. Try normal login
  final resp = await api.post('/auth/login', body: {
    'firebase_id_token': firebaseIdToken,
    'device_id': deviceId,
  });
  if (resp['status'] == 'ok') return true;
  if (resp['status'] != 'device_conflict') return false;

  // 2. Ask user to confirm switch
  if (!await showSwitchDeviceDialog()) {
    await FirebaseAuth.instance.signOut();
    return false;
  }

  // 3. Force-login → returns custom_token
  final force = await api.post('/auth/force-login', body: {
    'firebase_id_token': firebaseIdToken,
    'device_id': deviceId,
  });
  if (force['status'] != 'ok') return false;

  return _completeWithCustomToken(force['custom_token'], deviceId);
}

/// For Phone+Password — backend gives custom_token directly.
Future<bool> handlePhoneLogin(String phone, String password, String deviceId) async {
  // 1. Try normal phone-login
  var resp = await api.post('/auth/phone-login', body: {
    'phone': phone,
    'password': password,
    'device_id': deviceId,
  });

  // 2. Handle conflict by retrying with force=true
  if (resp['status'] == 'device_conflict') {
    if (!await showSwitchDeviceDialog()) return false;
    resp = await api.post('/auth/phone-login', body: {
      'phone': phone,
      'password': password,
      'device_id': deviceId,
      'force': true,
    });
  }

  if (resp['status'] != 'ok') return false;
  return _completeWithCustomToken(resp['custom_token'], deviceId);
}

/// Shared completion: exchange custom_token → Firebase session → /auth/login.
Future<bool> _completeWithCustomToken(String customToken, String deviceId) async {
  // Sign in to Firebase with the new token — fresh session, fresh refresh token
  final cred = await FirebaseAuth.instance.signInWithCustomToken(customToken);
  final freshIdToken = await cred.user?.getIdToken();
  if (freshIdToken == null) return false;

  // Tell your API client to use the new token going forward
  api.setIdToken(freshIdToken);

  // Register the session with the backend
  final loginResp = await api.post('/auth/login', body: {
    'firebase_id_token': freshIdToken,
    'device_id': deviceId,
  });
  return loginResp['status'] == 'ok';
}
```

### Flutter — 401 Interceptor (handles getting kicked from another device)

A user signed in on Device A may get force-logged-out when they sign in on
Device B. The next API call from Device A returns 401 and the refresh token
is gone. Catch this in a global interceptor.

```dart
Future<http.Response> authedRequest(
  String method, String path, {Map? body}
) async {
  var resp = await _send(method, path, body: body);
  if (resp.statusCode != 401) return resp;

  // Try to refresh the token
  String? freshToken;
  try {
    freshToken = await FirebaseAuth.instance.currentUser?.getIdToken(true);
  } catch (_) {
    freshToken = null;
  }

  if (freshToken == null) {
    // Refresh token revoked → user was kicked
    await FirebaseAuth.instance.signOut();
    await secureStorage.deleteAll(); // clear biometric_secret etc.
    navigateToLoginScreen();
    return resp;
  }

  api.setIdToken(freshToken);
  return _send(method, path, body: body); // retry once
}
```

---

## Swift — Universal Handler

```swift
import FirebaseAuth

/// For Google / Apple / Email — caller has already done Firebase SDK login.
func handleLogin(firebaseIdToken: String, deviceId: String) async throws -> Bool {
    // 1. Try normal login
    let resp = try await api.post("/auth/login", body: [
        "firebase_id_token": firebaseIdToken,
        "device_id": deviceId,
    ])
    if resp["status"] as? String == "ok" { return true }
    guard resp["status"] as? String == "device_conflict" else { return false }

    // 2. Confirm with user
    let confirmed = await showSwitchDeviceDialog()
    guard confirmed else {
        try? Auth.auth().signOut()
        return false
    }

    // 3. Force-login → returns custom_token
    let force = try await api.post("/auth/force-login", body: [
        "firebase_id_token": firebaseIdToken,
        "device_id": deviceId,
    ])
    guard force["status"] as? String == "ok",
          let customToken = force["custom_token"] as? String else { return false }

    return try await completeWithCustomToken(customToken, deviceId: deviceId)
}

/// For Phone+Password — backend returns custom_token directly.
func handlePhoneLogin(phone: String, password: String, deviceId: String) async throws -> Bool {
    var resp = try await api.post("/auth/phone-login", body: [
        "phone": phone,
        "password": password,
        "device_id": deviceId,
    ])

    if resp["status"] as? String == "device_conflict" {
        let confirmed = await showSwitchDeviceDialog()
        guard confirmed else { return false }
        resp = try await api.post("/auth/phone-login", body: [
            "phone": phone,
            "password": password,
            "device_id": deviceId,
            "force": true,
        ])
    }

    guard resp["status"] as? String == "ok",
          let customToken = resp["custom_token"] as? String else { return false }

    return try await completeWithCustomToken(customToken, deviceId: deviceId)
}

/// Shared completion: exchange custom_token → Firebase session → /auth/login.
private func completeWithCustomToken(
    _ customToken: String, deviceId: String
) async throws -> Bool {
    let cred = try await Auth.auth().signIn(withCustomToken: customToken)
    let freshIdToken = try await cred.user.getIDToken()
    api.setIdToken(freshIdToken)

    let loginResp = try await api.post("/auth/login", body: [
        "firebase_id_token": freshIdToken,
        "device_id": deviceId,
    ])
    return loginResp["status"] as? String == "ok"
}
```

### Swift — 401 Interceptor

```swift
func authedRequest(_ request: URLRequest) async throws -> Data {
    var (data, resp) = try await URLSession.shared.data(for: request)
    guard (resp as? HTTPURLResponse)?.statusCode == 401 else { return data }

    let fresh: String?
    do { fresh = try await Auth.auth().currentUser?.getIDTokenForcingRefresh(true) }
    catch { fresh = nil }

    guard let freshToken = fresh else {
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

---

## What Each Endpoint Returns

| Auth method | Conflict on | Force endpoint | Force response |
|---|---|---|---|
| Google | `/auth/login` | `/auth/force-login` | `custom_token` |
| Apple | `/auth/login` | `/auth/force-login` | `custom_token` |
| Email + Password | `/auth/login` | `/auth/force-login` | `custom_token` |
| Phone + Password | `/auth/phone-login` | `/auth/phone-login(force=true)` | `custom_token` |

All four converge on the same client step: `signInWithCustomToken(custom_token)`.

---

## Why the Custom Token Approach

1. **`/auth/force-login` calls `firebase_auth.revoke_refresh_tokens(uid)`.** This invalidates every refresh token Firebase has issued for that user — including the one the calling device holds.
2. **The `firebase_id_token` you sent is also invalidated** by the backend's `revoked_before` Redis key (the dependency check in `deps.py` rejects any token with `iat < revoked_before`).
3. **`getIdToken(forceRefresh: true)` would throw** because the refresh token is gone.
4. **The only way forward is to mint a new session.** A custom token created *after* the revocation has `auth_time > tokensValidAfterTime` and produces a brand-new Firebase session with a brand-new refresh token.

This is the same mechanism `signInWithCustomToken` is designed for. Once the
client signs in with the custom token, everything downstream (idToken refresh,
etc.) works automatically — no special handling, no re-authentication prompts.

---

## Common Mistakes to Avoid

- ❌ **Don't** call `getIdToken(forceRefresh: true)` immediately after `/auth/force-login` — the refresh token is gone, it will throw.
- ❌ **Don't** retry the same `firebase_id_token` after force-login — the backend will reject it as revoked.
- ❌ **Don't** prompt the user to re-enter their password / re-OAuth — not necessary, the custom_token gives you a fresh session.
- ❌ **Don't** forget to update your API client's stored token after `signInWithCustomToken`. The old token is dead.
- ✅ **Do** add a global 401 interceptor — devices that get kicked from elsewhere will see 401s and need to fall back to manual login.
