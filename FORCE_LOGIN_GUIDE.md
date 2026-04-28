# Force Login (Device Switch) — Implementation Guide

When a user tries to log in on a new device while already active on another device,
the backend returns `{"status": "device_conflict"}`. The user must explicitly confirm
they want to switch devices. This guide covers how to handle that for each auth method.

---

## Google / Apple / Email+Password

These three providers go through Firebase SDK first, so the app already has a
`firebase_id_token` by the time the conflict is detected.

### Flow

```
1. Firebase SDK login → app has firebase_id_token
2. POST /auth/login(firebase_id_token, device_id) → device_conflict
3. Show dialog: "You're signed in on another device. Switch here?"
4. User taps "Switch Device"
5. POST /auth/force-login(same firebase_id_token, device_id) → {"status": "ok"}
6. Proceed to app normally
```

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

### Flutter Example

```dart
Future<void> handleLogin(String firebaseIdToken, String deviceId) async {
  final resp = await api.post('/auth/login', body: {
    'firebase_id_token': firebaseIdToken,
    'device_id': deviceId,
  });

  if (resp['status'] == 'device_conflict') {
    final confirmed = await showSwitchDeviceDialog();
    if (!confirmed) return;

    await api.post('/auth/force-login', body: {
      'firebase_id_token': firebaseIdToken, // same token — still valid
      'device_id': deviceId,
    });

    // Proceed to app
  }
}
```

### Swift Example

```swift
func handleLogin(firebaseIdToken: String, deviceId: String) async throws {
    let resp = try await api.post("/auth/login", body: [
        "firebase_id_token": firebaseIdToken,
        "device_id": deviceId,
    ])

    if resp["status"] as? String == "device_conflict" {
        let confirmed = await showSwitchDeviceDialog()
        guard confirmed else { return }

        try await api.post("/auth/force-login", body: [
            "firebase_id_token": firebaseIdToken, // same token — still valid
            "device_id": deviceId,
        ])

        // Proceed to app
    }
}
```

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
