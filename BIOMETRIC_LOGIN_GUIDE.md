# Biometric Login — Implementation Guide

A complete walkthrough for enabling fingerprint / Face ID login on top of Firebase Auth.

This guide replaces any earlier instructions that involved storing a Firebase **refresh token**. That approach does not work — `FirebaseAuth.instance.currentUser?.refreshToken` returns `null` on iOS and Android in FlutterFire ([issue #8171](https://github.com/firebase/flutterfire/issues/8171)). Do not store the refresh token.

---

## How It Works

The backend issues a one-time random `biometric_secret` when the user enables biometric. The plaintext is returned **only once**; only its SHA-256 hash is stored on the server. The client keeps the plaintext in secure storage gated behind biometric.

On future launches, the user passes biometric → the app reads the secret → backend exchanges it for a Firebase **custom token** → app signs in to Firebase → calls `/auth/login` to register the session.

```
ENABLE (one time, user already logged in):

  POST /auth/enable-biometric
  ──────────────────────────────────────►
                                          │
                                          ▼
                                   Backend generates secret,
                                   stores SHA-256 hash, returns
                                   plaintext to client.
  ◄──────────────────────────────────────
  { biometric_secret: "kya_..." }

  Client stores secret in Keychain / Keystore behind biometric.


LOGIN (every app launch after that):

  Biometric prompt → user passes
                │
                ▼
  Read secret from secure storage
                │
                ▼
  POST /auth/biometric-login { secret, device_id }
  ──────────────────────────────────────►
                                          │
                                          ▼
                                   Backend hashes secret, finds user,
                                   mints Firebase custom_token.
  ◄──────────────────────────────────────
  { custom_token: "..." }
                │
                ▼
  FirebaseAuth.signInWithCustomToken(custom_token)
                │
                ▼
  currentUser.getIdToken() → idToken
                │
                ▼
  POST /auth/login { idToken, device_id }
                │
                ▼
            User is in
```

---

## Endpoints

### `POST /auth/enable-biometric`

**Auth:** Standard headers (`Authorization: Bearer <id_token>` + `X-Device-Id`)

**Response (200):**
```json
{
  "biometric_enabled": true,
  "biometric_secret": "kya_a1b2c3..."
}
```

Calling this again rotates the secret — old secret stops working.

---

### `POST /auth/biometric-login`

**Auth:** None — the secret is the credential.

**Body:**
```json
{
  "biometric_secret": "kya_...",
  "device_id": "iphone-abc-123",
  "force": false
}
```

**Response (200) — Success:**
```json
{
  "status": "ok",
  "custom_token": "eyJhbG...",
  "user_id": "550e8400-..."
}
```

**Response (200) — Device conflict:**
```json
{
  "status": "device_conflict",
  "message": "Active session on another device. Use force=true to switch."
}
```

**Errors:**
- `401` — Invalid biometric credential (wrong secret, or user disabled biometric)
- `403` — Account suspended

---

### `POST /auth/disable-biometric`

**Auth:** Standard headers.

**Response (200):**
```json
{ "biometric_enabled": false }
```

Server-side hash is cleared. The app should also delete the secret from secure storage.

---

## Flutter Implementation

```dart
import 'package:firebase_auth/firebase_auth.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:local_auth/local_auth.dart';

final localAuth = LocalAuthentication();
final secureStorage = const FlutterSecureStorage();

const _kBiometricSecretKey = 'biometric_secret';

// ─── Enable biometric (called from settings screen) ───
Future<void> enableBiometric() async {
  final resp = await api.post('/auth/enable-biometric');
  final secret = resp['biometric_secret'] as String;

  await secureStorage.write(
    key: _kBiometricSecretKey,
    value: secret,
    aOptions: const AndroidOptions(encryptedSharedPreferences: true),
    iOptions: const IOSOptions(accessibility: KeychainAccessibility.passcode),
  );
}

// ─── Disable biometric ───
Future<void> disableBiometric() async {
  await api.post('/auth/disable-biometric');
  await secureStorage.delete(key: _kBiometricSecretKey);
}

// ─── Check if biometric is set up on this device ───
Future<bool> isBiometricAvailable() async {
  final hasSecret = await secureStorage.read(key: _kBiometricSecretKey) != null;
  final canCheck = await localAuth.canCheckBiometrics;
  return hasSecret && canCheck;
}

// ─── Login with biometric (called on app launch / login screen) ───
Future<bool> biometricLogin(String deviceId) async {
  // 1. Local biometric prompt
  final authed = await localAuth.authenticate(
    localizedReason: 'Sign in to Kaya',
    options: const AuthenticationOptions(biometricOnly: true),
  );
  if (!authed) return false;

  // 2. Read the secret
  final secret = await secureStorage.read(key: _kBiometricSecretKey);
  if (secret == null) return false;

  // 3. Exchange secret for Firebase custom token
  var resp = await api.post('/auth/biometric-login', body: {
    'biometric_secret': secret,
    'device_id': deviceId,
  });

  // 4. Handle device conflict
  if (resp['status'] == 'device_conflict') {
    final confirmed = await showSwitchDeviceDialog();
    if (!confirmed) return false;
    resp = await api.post('/auth/biometric-login', body: {
      'biometric_secret': secret,
      'device_id': deviceId,
      'force': true,
    });
  }

  if (resp['status'] != 'ok') {
    // Secret was rejected — user must log in manually and re-enable biometric
    await secureStorage.delete(key: _kBiometricSecretKey);
    return false;
  }

  // 5. Sign in to Firebase with the custom token
  final cred = await FirebaseAuth.instance.signInWithCustomToken(resp['custom_token']);
  final idToken = await cred.user?.getIdToken();

  // 6. Register the session with backend
  final loginResp = await api.post('/auth/login', body: {
    'firebase_id_token': idToken,
    'device_id': deviceId,
  });
  return loginResp['status'] == 'ok';
}
```

---

## Swift Implementation

```swift
import FirebaseAuth
import LocalAuthentication
import Security

private let kBiometricSecretAccount = "biometric_secret"

// ─── Enable biometric ───
func enableBiometric() async throws {
    let resp = try await api.post("/auth/enable-biometric")
    guard let secret = resp["biometric_secret"] as? String else {
        throw AuthError.invalidResponse
    }

    let access = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        .biometryCurrentSet,
        nil
    )!

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: kBiometricSecretAccount,
        kSecValueData as String: secret.data(using: .utf8)!,
        kSecAttrAccessControl as String: access,
    ]
    SecItemDelete(query as CFDictionary)
    SecItemAdd(query as CFDictionary, nil)
}

// ─── Disable biometric ───
func disableBiometric() async throws {
    try await api.post("/auth/disable-biometric")
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: kBiometricSecretAccount,
    ]
    SecItemDelete(query as CFDictionary)
}

// ─── Login with biometric ───
func biometricLogin(deviceId: String) async throws -> Bool {
    let context = LAContext()
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil)
    else { return false }

    // 1. Local biometric prompt
    try await context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: "Sign in to Kaya"
    )

    // 2. Read the secret from Keychain
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: kBiometricSecretAccount,
        kSecReturnData as String: true,
    ]
    var item: AnyObject?
    guard SecItemCopyMatching(query as CFDictionary, &item) == errSecSuccess,
          let data = item as? Data,
          let secret = String(data: data, encoding: .utf8)
    else { return false }

    // 3. Exchange secret for Firebase custom token
    var resp = try await api.post("/auth/biometric-login", body: [
        "biometric_secret": secret,
        "device_id": deviceId,
    ])

    // 4. Handle device conflict
    if resp["status"] as? String == "device_conflict" {
        let confirmed = await showSwitchDeviceDialog()
        guard confirmed else { return false }
        resp = try await api.post("/auth/biometric-login", body: [
            "biometric_secret": secret,
            "device_id": deviceId,
            "force": true,
        ])
    }

    guard resp["status"] as? String == "ok",
          let customToken = resp["custom_token"] as? String
    else {
        // Secret rejected — clear it from Keychain
        SecItemDelete(query as CFDictionary)
        return false
    }

    // 5. Sign in to Firebase
    let cred = try await Auth.auth().signIn(withCustomToken: customToken)
    let idToken = try await cred.user.getIDToken()

    // 6. Register session with backend
    let loginResp = try await api.post("/auth/login", body: [
        "firebase_id_token": idToken,
        "device_id": deviceId,
    ])
    return loginResp["status"] as? String == "ok"
}
```

---

## Common Scenarios

| Scenario | Behavior |
|---|---|
| User opens app for first time after enabling biometric | Biometric prompt → success → signed in |
| App was killed and reopened | Same as above |
| Token expired in background (60 min) | Firebase SDK refreshes it automatically — no biometric needed |
| User logged in on another device since last visit | `device_conflict` → show switch dialog → retry with `force: true` |
| User changes Face ID / re-enrolls fingerprint | Keychain entry invalidates → biometric login fails → fall back to manual login → user can re-enable |
| User disables biometric on this device | `/auth/disable-biometric` clears server hash; app deletes local secret |
| User reinstalls app | Secure storage is wiped → user must log in manually and re-enable |
| User enables biometric on a second device | Backend rotates the secret — old device's secret stops working. Only the latest enrolled device retains biometric. |

---

## What NOT To Do

- ❌ Do **not** store `currentUser?.refreshToken` — it is `null` on iOS/Android in FlutterFire.
- ❌ Do **not** store the Firebase ID token — it expires in 60 minutes.
- ❌ Do **not** call `signInWithCustomToken` on a refresh token — different token types.
- ❌ Do **not** cache the `biometric_secret` outside secure storage.
- ❌ Do **not** send the `biometric_secret` in the `Authorization` header — it goes in the request body.

---

## Security Notes

- The `biometric_secret` is **256 bits** of cryptographic randomness. Brute force is infeasible.
- The server stores **only the SHA-256 hash**. A DB leak does not expose any secrets.
- Secure storage is gated behind biometric — an attacker who steals the device but cannot pass biometric cannot extract the secret.
- The single-device rule still applies. An attacker on Device X cannot silently take over a session active on Device Y — they hit `device_conflict` and must explicitly force-switch (which kicks the legitimate user, who immediately notices on next request).
