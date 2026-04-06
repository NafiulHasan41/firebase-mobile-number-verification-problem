# Kaya Backend — API Documentation (M2)

> Base URL: `http://165.232.55.80:8000`
> Swagger UI: `http://165.232.55.80:8000/docs` (password protected)

> **Important — Streaming Required for Session & Chat:**
> All session messaging (`POST /api/v1/sessions/{id}/message`) and chat messaging (`POST /api/v1/chat/threads/{id}/message`) **must use the `/stream` endpoints**.
> - Session: `POST /api/v1/sessions/{session_id}/message/stream`
> - Chat: `POST /api/v1/chat/threads/{thread_id}/message/stream`
>
> The non-streaming variants exist but are not recommended — they block for 10–15 seconds with no feedback.
> The streaming endpoints use **Server-Sent Events (SSE)**. See `API_DOCS_CHAT_SESSION.md` for the full SSE event reference.

# Docs (HTTP Basic Auth for /docs, /redoc, /openapi.json)
>DOCS_USERNAME=admin
>DOCS_PASSWORD=KayaDocs2026!Pab%3&r>

---

## Firebase Setup (Mobile)

Before integrating with the API, you need the Firebase configuration files for the Kaya project.

### Required Files

| Platform | File | How to get |
|----------|------|------------|
| iOS | `GoogleService-Info.plist` | Firebase Console → Project Settings → Your apps → iOS → Download |
| Android | `google-services.json` | Firebase Console → Project Settings → Your apps → Android → Download |

Add these files to your project as per Firebase SDK docs:
- **iOS:** Place `GoogleService-Info.plist` in the Xcode project root
- **Android:** Place `google-services.json` in `app/` directory

### Enabled Auth Providers

The following sign-in methods are enabled in Firebase. Initialize them in your app:

| Provider | Firebase SDK Method |
|----------|-------------------|
| Google | `GoogleAuthProvider` |
| Apple | `OAuthProvider("apple.com")` |
| Email + Password | `createUserWithEmailAndPassword` / `signInWithEmailAndPassword` |
| Phone + OTP | `verifyPhoneNumber` (SMS verification) |

### Firebase SDK Initialization

```dart
// Flutter
await Firebase.initializeApp();
```

```swift
// iOS
FirebaseApp.configure()
```

```kotlin
// Android — auto-initialized via google-services.json
```

### What You Need From the Backend Team

1. The **Firebase config files** listed above (request from backend team)
2. The **base URL** of the deployed server
3. This API documentation

### What You Do NOT Need

- No API keys for the backend (removed — Firebase auth is the only security layer)
- No service account files (backend-only)
- No `.env` values

---

## Authentication

All API endpoints (except `/health` and auth signup/login) require Firebase authentication.

### Required Headers

Every authenticated request must include:

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <unique-device-identifier>
Content-Type: application/json
```

- `Authorization` — Firebase ID token obtained from Firebase client SDK after login (Google, Apple, Email, Phone)
- `X-Device-Id` — A unique, persistent identifier for the device (e.g. `UIDevice.current.identifierForVendor` on iOS, `Settings.Secure.ANDROID_ID` on Android)

### Token Refresh

Firebase ID tokens expire every **60 minutes**. The mobile app must implement a 401 retry interceptor:

```
1. Make API call
2. If 401 response → call Firebase SDK getIdToken(forceRefresh: true)
3. Retry the request with the fresh token
```

### Single Device Session

Only one device can be active per user at a time. If a user logs in on a new device:
- The API returns `{"status": "device_conflict"}` on login
- The user can call `/auth/force-login` to switch devices (kicks the old one)
- The old device's token is revoked

#### Device Conflict — UI Flow

```
1. User opens app on Device B
2. App calls POST /auth/login → gets {"status": "device_conflict"}
3. App shows dialog:
   ┌─────────────────────────────────────┐
   │  Active on another device           │
   │                                     │
   │  You're currently signed in on      │
   │  another device. Sign in here       │
   │  and sign out everywhere else?      │
   │                                     │
   │  [Cancel]           [Switch Device] │
   └─────────────────────────────────────┘
4. User taps "Switch Device"
5. App calls POST /auth/force-login → {"status": "ok"}
6. Device B is now active, Device A is kicked out
7. On Device A: next API call gets 401 → show login screen
```

If user taps "Cancel", stay on login screen. Do NOT retry `/auth/login` — the device conflict still exists.

### New User vs Returning User

The app needs to determine whether to call `/auth/signup` or `/auth/login`:

```
1. User authenticates via Firebase (Google/Apple/Email/Phone)
2. Get Firebase ID token
3. Call POST /auth/login first
4. If 200 → user exists, logged in
5. If 404 "No account found" → user is new
6. Call POST /auth/signup → account created
```

Always try login first. Signup returns 409 if the user already exists, and login returns 404 if they don't. Login-first avoids the 409 error on returning users.

**Note:** Signup does NOT return `biometric_enabled` in the response (it's always false for new accounts). Login DOES return it. After signup, you can assume `biometric_enabled: false`.

### Signup Flows by Auth Method

Each auth method has a different signup flow. The backend enforces verification before allowing access.

#### Google / Apple Signup

No verification needed — OAuth providers already verify identity.

```
1. User taps "Sign in with Google" or "Sign in with Apple"
2. Firebase client SDK handles OAuth → returns ID token
3. Call POST /auth/login → 404 (new user)
4. Call POST /auth/signup { firebase_id_token, name, device_id }
5. Response: { "status": "ok", "verification_required": false }
6. User is in → proceed to app
```

#### Email + Password Signup (Verification Required)

Users must verify their email before they can use the app. The backend blocks all API access until verified.

```
1. User enters email + password
2. App calls Firebase createUserWithEmailAndPassword(email, password)
3. App calls Firebase sendEmailVerification() → Firebase sends email with link
4. App calls POST /auth/signup { firebase_id_token, name, device_id }
5. Response: { "status": "ok", "verification_required": true,
               "message": "Verification email sent. Please check your inbox." }

6. Show "Check Your Email" screen:
   ┌─────────────────────────────────────┐
   │  Verify your email                  │
   │                                     │
   │  We sent a verification link to     │
   │  user@example.com                   │
   │                                     │
   │  Click the link in the email,       │
   │  then tap "I've Verified" below.    │
   │                                     │
   │  [I've Verified]                    │
   │                                     │
   │  Didn't receive it? [Resend Email]  │
   └─────────────────────────────────────┘

7. User clicks link in email → Firebase marks email_verified = true

8. User taps "I've Verified":
   - App calls Firebase getIdToken(forceRefresh: true) → fresh token
   - App calls POST /auth/login { firebase_id_token, device_id }
   - If 200 → verified, proceed to app
   - If 403 "verify email" → not verified yet, show message

9. User taps "Resend Email":
   - App calls POST /auth/resend-verification { firebase_id_token }
   - Show: "Verification email sent"
```

**What happens if the user tries to use the app without verifying:**
- `POST /auth/login` → 403 "Please verify your email. Check your inbox."
- Any protected endpoint (sessions, chat, etc.) → 403 "Please verify your email before continuing."

**Flutter example — Email signup:**
```dart
// Step 1-3: Firebase signup + send verification
final credential = await FirebaseAuth.instance
    .createUserWithEmailAndPassword(email: email, password: password);
await credential.user!.sendEmailVerification();
final idToken = await credential.user!.getIdToken();

// Step 4: Register with backend
final resp = await api.post('/auth/signup', body: {
  'firebase_id_token': idToken,
  'name': name,
  'device_id': deviceId,
});

if (resp['verification_required'] == true) {
  // Show "Check your email" screen
  Navigator.push(context, VerifyEmailScreen());
}

// Step 8: After user says they verified
Future<bool> checkVerification() async {
  final freshToken = await FirebaseAuth.instance.currentUser?.getIdToken(true);
  try {
    await api.post('/auth/login', body: {
      'firebase_id_token': freshToken,
      'device_id': deviceId,
    });
    return true; // Verified — proceed to app
  } catch (e) {
    return false; // Not verified yet
  }
}

// Step 9: Resend
Future<void> resendVerification() async {
  final token = await FirebaseAuth.instance.currentUser?.getIdToken();
  await api.post('/auth/resend-verification', body: {
    'firebase_id_token': token,
  });
}
```

**Swift/iOS example — Email signup:**
```swift
// Step 1-3: Firebase signup + send verification
let result = try await Auth.auth().createUser(withEmail: email, password: password)
try await result.user.sendEmailVerification()
let idToken = try await result.user.getIDToken()

// Step 4: Register with backend
let resp = try await api.post("/auth/signup", body: [
    "firebase_id_token": idToken,
    "name": name,
    "device_id": deviceId,
])

if resp["verification_required"] as? Bool == true {
    // Show "Check your email" screen
}

// Step 8: After user says they verified
func checkVerification() async -> Bool {
    let freshToken = try? await Auth.auth().currentUser?.getIDTokenForcingRefresh(true)
    do {
        try await api.post("/auth/login", body: [
            "firebase_id_token": freshToken ?? "",
            "device_id": deviceId,
        ])
        return true // Verified
    } catch {
        return false // Not yet
    }
}

// Step 9: Resend
func resendVerification() async {
    let token = try? await Auth.auth().currentUser?.getIDToken()
    try? await api.post("/auth/resend-verification", body: [
        "firebase_id_token": token ?? "",
    ])
}
```

#### Phone + Password Signup (OTP + Password Required)

Firebase handles SMS OTP verification. After phone is verified, the user must set a password. The backend requires the password.

```
1. User enters phone number

2. App calls Firebase verifyPhoneNumber(phoneNumber):
   - Firebase sends SMS with 6-digit code
   - User receives SMS

3. Show "Enter Code" screen:
   ┌─────────────────────────────────────┐
   │  Enter verification code            │
   │                                     │
   │  We sent a 6-digit code to          │
   │  +1 (234) 567-8900                  │
   │                                     │
   │  [ _ ] [ _ ] [ _ ] [ _ ] [ _ ] [ _ ]│
   │                                     │
   │  [Verify]                           │
   │                                     │
   │  Didn't receive it? [Resend Code]   │
   └─────────────────────────────────────┘

4. User enters 6-digit code
5. Firebase verifies → returns credential
6. App calls Firebase signInWithCredential(credential) → returns ID token

7. Show "Set Password" screen:
   ┌─────────────────────────────────────┐
   │  Create a password                  │
   │                                     │
   │  This will be used for future       │
   │  logins (no SMS needed).            │
   │                                     │
   │  Password: [________]               │
   │  Confirm:  [________]               │
   │                                     │
   │  Min 8 characters                   │
   │                                     │
   │  [Continue]                         │
   └─────────────────────────────────────┘

8. App calls POST /auth/signup:
   {
     "firebase_id_token": "<token from step 6>",
     "name": "User Name",
     "password": "MyPass123",     ← REQUIRED for phone users
     "device_id": "device-id"
   }

9. Response: { "status": "ok", "verification_required": false }
10. User is in → proceed to app
```

**What happens if password is not provided:**
- `POST /auth/signup` → 400 "Password is required for phone signup."

**Future logins — phone + password only (no SMS):**
```
1. User enters phone number + password
2. App calls POST /auth/phone-login { phone, password, device_id }
3. Response: { "custom_token": "..." }
4. App calls Firebase signInWithCustomToken(customToken) → ID token
5. Use ID token for all API calls
```

**Flutter example — Phone signup:**
```dart
// Step 2: Start phone verification
await FirebaseAuth.instance.verifyPhoneNumber(
  phoneNumber: phoneNumber,
  verificationCompleted: (credential) async {
    // Auto-verification (Android) — sign in directly
    await FirebaseAuth.instance.signInWithCredential(credential);
  },
  verificationFailed: (e) {
    showError(e.message ?? 'Verification failed');
  },
  codeSent: (verificationId, resendToken) {
    // Show "Enter Code" screen
    Navigator.push(context, EnterCodeScreen(verificationId: verificationId));
  },
  codeAutoRetrievalTimeout: (_) {},
);

// Step 4-6: User enters code
final credential = PhoneAuthProvider.credential(
  verificationId: verificationId,
  smsCode: userEnteredCode,
);
await FirebaseAuth.instance.signInWithCredential(credential);
final idToken = await FirebaseAuth.instance.currentUser?.getIdToken();

// Step 7-8: Show password screen, then signup
final resp = await api.post('/auth/signup', body: {
  'firebase_id_token': idToken,
  'name': name,
  'password': password,  // REQUIRED
  'device_id': deviceId,
});
// Proceed to app

// Future logins:
final loginResp = await api.post('/auth/phone-login', body: {
  'phone': phoneNumber,
  'password': password,
  'device_id': deviceId,
});
final customToken = loginResp['custom_token'];
await FirebaseAuth.instance.signInWithCustomToken(customToken);
```

### Resend Verification Email

```
POST /api/v1/auth/resend-verification
```

For email users who haven't verified yet. Triggers Firebase to send a new verification email.

**Body:**
```json
{
  "firebase_id_token": "eyJhbG..."
}
```

**Response (200):**
```json
{
  "status": "ok",
  "message": "Verification email sent. Please check your inbox."
}
```

**Errors:**
- `401` — Invalid token
- `400` — Not an email account / Already verified

**Rate limit:** 3/minute

### Apple Sign-In

Apple Sign-In requires a **nonce** for security. The app must generate a random nonce, hash it with SHA256, and pass it to Apple. Firebase handles this, but make sure to use the Firebase-specific Apple auth flow:

```dart
// Flutter — Apple Sign-In with Firebase
import 'package:sign_in_with_apple/sign_in_with_apple.dart';
import 'dart:convert';
import 'dart:math';
import 'package:crypto/crypto.dart';

String generateNonce([int length = 32]) {
  final random = Random.secure();
  return List.generate(length, (_) => random.nextInt(256).toRadixString(16).padLeft(2, '0')).join();
}

Future<String> signInWithApple() async {
  final rawNonce = generateNonce();
  final hashedNonce = sha256.convert(utf8.encode(rawNonce)).toString();

  final credential = await SignInWithApple.getAppleIDCredential(
    scopes: [AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName],
    nonce: hashedNonce,
  );

  final oauthCredential = OAuthProvider("apple.com").credential(
    idToken: credential.identityToken,
    rawNonce: rawNonce,
  );

  final userCredential = await FirebaseAuth.instance.signInWithCredential(oauthCredential);
  return await userCredential.user!.getIdToken() ?? '';
  // Then call /auth/login or /auth/signup with this token
}
```

```swift
// Swift — Apple Sign-In with Firebase
import AuthenticationServices
import CryptoKit

func startAppleSignIn() {
    let nonce = randomNonceString()
    let hashedNonce = SHA256.hash(data: Data(nonce.utf8))
        .compactMap { String(format: "%02x", $0) }.joined()

    let request = ASAuthorizationAppleIDProvider().createRequest()
    request.requestedScopes = [.fullName, .email]
    request.nonce = hashedNonce

    let controller = ASAuthorizationController(authorizationRequests: [request])
    controller.delegate = self
    controller.performRequests()
    // Store nonce for Firebase credential creation in the delegate callback
}

// In delegate callback:
func authorizationController(controller: ASAuthorizationController,
                             didCompleteWithAuthorization authorization: ASAuthorization) {
    guard let appleIDCredential = authorization.credential as? ASAuthorizationAppleIDCredential,
          let identityToken = appleIDCredential.identityToken,
          let tokenString = String(data: identityToken, encoding: .utf8) else { return }

    let credential = OAuthProvider.credential(
        providerID: .apple,
        idToken: tokenString,
        rawNonce: currentNonce  // The unhashed nonce from startAppleSignIn
    )

    Auth.auth().signIn(with: credential) { result, error in
        guard let user = result?.user else { return }
        user.getIDToken { idToken, _ in
            // Call /auth/login or /auth/signup with idToken
        }
    }
}
```

**Apple-specific requirements:**
- You must configure "Sign in with Apple" in Apple Developer portal and link it in Firebase Console
- Apple only returns the user's name on the **first** sign-in. Store it immediately via `/auth/signup` with the `name` field
- Apple may return `nil` for email (users can choose to hide it). Firebase assigns a relay email like `abc@privaterelay.appleid.com`

### Session Expiry Handling

Coaching sessions have a 24-hour timer. If the session expires while the user is chatting:

```
1. User sends a message
2. App calls POST /sessions/{id}/message/stream
3. Backend checks timer → expired
4. Returns 400: {"detail": "Session has expired"}
```

**How the app should handle this:**

```
1. On 400 "Session has expired":
   - Show message: "This session has expired after 24 hours."
   - Disable the message input
   - Show "View Messages" button (GET /sessions/{id}/messages — read-only)
   - Show "Start New Session" button

2. Proactive timer check:
   - GET /sessions/active returns time_remaining_seconds
   - Show countdown timer in the UI
   - When timer reaches 0, disable input without waiting for 400

3. Session history:
   - Expired sessions are still readable (GET /sessions/{id}/messages)
   - User can view all past messages but cannot send new ones
   - AI memory from the expired session is preserved
```

**Timer display logic:**
```dart
// Flutter — session timer
int timeRemaining = activeSession['time_remaining_seconds'];

String formatTimer(int seconds) {
  final hours = seconds ~/ 3600;
  final minutes = (seconds % 3600) ~/ 60;
  return '${hours}h ${minutes}m remaining';
}

// Poll every 60 seconds to keep timer accurate
Timer.periodic(Duration(seconds: 60), (_) async {
  final resp = await api.get('/sessions/active');
  final active = resp['active_session'];
  if (active == null) {
    // Session expired — disable input
    showSessionExpiredDialog();
  } else {
    timeRemaining = active['time_remaining_seconds'];
    updateTimerUI(formatTimer(timeRemaining));
  }
});
```

### Swagger UI Access

The API documentation at `/docs` is password protected:

```
Username: admin
Password: (set in .env as DOCS_PASSWORD)
```

Browser will show a native login popup.

---

## Auth Endpoints

> These endpoints handle signup, login, device management, and password flows. The signup/login endpoints verify the Firebase token from the request body — they do NOT require the `Authorization` header.

### 1. Sign Up

```
POST /api/v1/auth/signup
```

**Body:**
```json
{
  "firebase_id_token": "eyJhbG...",
  "name": "Sarah",
  "device_id": "iphone-abc-123",
  "password": "MyPass123"
}
```

- `name` — optional (can also come from Firebase profile)
- `password` — required only for phone+password users (min 8 chars)

**Response (200) — Google / Apple / Phone (no verification needed):**
```json
{
  "status": "ok",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "firebase_uid": "nfvCkbGQlBcDJgBoPNEIdd4CyaY2",
  "provider": "google",
  "verification_required": false,
  "message": "Account created."
}
```

**Response (200) — Email (verification required):**
```json
{
  "status": "ok",
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "firebase_uid": "nfvCkbGQlBcDJgBoPNEIdd4CyaY2",
  "provider": "email",
  "verification_required": true,
  "message": "Verification email sent. Please check your inbox."
}
```

`provider` — one of: `google`, `apple`, `email`, `phone`

`verification_required` — `true` for email users who haven't verified yet, `false` for all others. **Always check this field after signup** — if `true`, show the "Check Your Email" screen instead of proceeding to the app.

**Errors:**
- `401` — Invalid Firebase token
- `409` — User already exists (use `/auth/login`)
- `400` — Password too short (phone+password only)
- `429` — Rate limit exceeded → returns **plain text** (not JSON), e.g. `"Rate limit exceeded: 5 per 1 minute"`. **Must guard against this separately before calling `jsonDecode`.**

**Rate limit:** 5/minute

---

### 2. Login

```
POST /api/v1/auth/login
```

**Body:**
```json
{
  "firebase_id_token": "eyJhbG...",
  "device_id": "iphone-abc-123"
}
```

**Response (200) — Success:**
```json
{
  "status": "ok",
  "user_id": "550e8400-...",
  "name": "Sarah",
  "biometric_enabled": false
}
```

**Response (200) — Device conflict:**
```json
{
  "status": "device_conflict",
  "message": "Active session on another device. Use /auth/force-login to switch."
}
```

**Errors:**
- `401` — Invalid Firebase token
- `404` — No account found
- `403` — Account suspended
- `403` — Email not verified → `"Please verify your email. Check your inbox."` (show resend email screen)
- `429` — Rate limit exceeded → **plain text** body, not JSON. Guard before `jsonDecode`.

**Rate limit:** 10/minute

---

### 3. Force Login (Switch Devices)

```
POST /api/v1/auth/force-login
```

Kicks the old device and registers the new one. Revokes all existing tokens.

**Body:**
```json
{
  "firebase_id_token": "eyJhbG...",
  "device_id": "samsung-xyz-456"
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

**Rate limit:** 5/minute

---

### 4. Phone+Password Login

```
POST /api/v1/auth/phone-login
```

For users who signed up with phone number + password. Returns a Firebase custom token the app exchanges for an ID token.

**Body:**
```json
{
  "phone": "+1234567890",
  "password": "MyPass123",
  "device_id": "iphone-abc-123"
}
```

**Response (200):**
```json
{
  "status": "ok",
  "custom_token": "eyJhbG...",
  "user_id": "550e8400-..."
}
```

The app must exchange `custom_token` for a Firebase ID token:
- **iOS:** `Auth.auth().signIn(withCustomToken: token)`
- **Android:** `Firebase.auth.signInWithCustomToken(token)`

Then use the resulting ID token for all subsequent API calls.

**Response (200) — Device conflict:**
```json
{
  "status": "device_conflict",
  "message": "Active session on another device. Use /auth/force-login."
}
```

**Errors:**
- `401` — Invalid phone number or password
- `403` — Account suspended

**Rate limit:** 5/minute

---

### 5. Logout

```
POST /api/v1/auth/logout
```

**Headers:**
```
Authorization: Bearer <firebase_id_token>
```

Note: Only `Authorization` header needed (no `X-Device-Id`).

**Response (200):**
```json
{
  "status": "logged_out"
}
```

Revokes all Firebase refresh tokens and invalidates the current session. Old tokens are immediately rejected.

---

### 6. Enable Biometric

```
POST /api/v1/auth/enable-biometric
```

**Headers:** Standard auth headers (`Authorization` + `X-Device-Id`)

**Response (200):**
```json
{
  "biometric_enabled": true
}
```

---

### 7. Disable Biometric

```
POST /api/v1/auth/disable-biometric
```

**Headers:** Standard auth headers

**Response (200):**
```json
{
  "biometric_enabled": false
}
```

#### Biometric Login — Implementation Guide

The backend stores a `biometric_enabled` flag. The actual biometric (Face ID / fingerprint) is handled entirely on the device. The backend never sees biometric data.

**Setup (after user enables biometric):**

```
1. User is logged in (has valid Firebase token)
2. User taps "Enable Biometric" in settings
3. App calls POST /auth/enable-biometric → backend sets flag to true
4. App stores Firebase refresh token in device secure storage,
   locked behind biometric:

   iOS:  Keychain with .biometryCurrentSet access control
   Android: Keystore with BiometricPrompt binding
```

**Login with biometric (next app launch):**

```
1. App launches
2. Check: did the last login response have biometric_enabled: true?
3. YES → prompt Face ID / fingerprint (local device operation)
4. Biometric succeeds → OS unlocks Keychain/Keystore
5. App reads stored Firebase refresh token
6. Firebase SDK exchanges refresh token for fresh ID token
7. App calls POST /auth/login with the fresh token + device_id
8. Normal auth flow from here
```

**Flutter example:**
```dart
import 'package:local_auth/local_auth.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';

final localAuth = LocalAuthentication();
final secureStorage = FlutterSecureStorage();

// After enabling biometric — store refresh token
Future<void> setupBiometric(String refreshToken) async {
  await api.post('/auth/enable-biometric');
  await secureStorage.write(
    key: 'firebase_refresh_token',
    value: refreshToken,
    aOptions: AndroidOptions(encryptedSharedPreferences: true),
    iOptions: IOSOptions(accessibility: KeychainAccessibility.passcode),
  );
}

// On app launch — biometric login
Future<bool> biometricLogin() async {
  final canAuth = await localAuth.canCheckBiometrics;
  if (!canAuth) return false;

  final authenticated = await localAuth.authenticate(
    localizedReason: 'Sign in to Kaya',
    options: const AuthenticationOptions(biometricOnly: true),
  );
  if (!authenticated) return false;

  // Biometric passed — get stored token
  final refreshToken = await secureStorage.read(key: 'firebase_refresh_token');
  if (refreshToken == null) return false;

  // Exchange for fresh ID token via Firebase SDK
  await FirebaseAuth.instance.signInWithCustomToken(refreshToken);
  final idToken = await FirebaseAuth.instance.currentUser?.getIdToken();

  // Login with backend
  final resp = await api.post('/auth/login', body: {
    'firebase_id_token': idToken,
    'device_id': deviceId,
  });

  return resp['status'] == 'ok';
}
```

**Swift/iOS example:**
```swift
import LocalAuthentication
import Security

// After enabling biometric — store refresh token in Keychain
func setupBiometric(refreshToken: String) async throws {
    try await api.post("/auth/enable-biometric")

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: "firebase_refresh_token",
        kSecValueData as String: refreshToken.data(using: .utf8)!,
        kSecAttrAccessControl as String: SecAccessControlCreateWithFlags(
            nil, .biometryCurrentSet, .privateKeyUsage, nil
        )!
    ]
    SecItemDelete(query as CFDictionary) // Remove old if exists
    SecItemAdd(query as CFDictionary, nil)
}

// On app launch — biometric login
func biometricLogin() async throws -> Bool {
    let context = LAContext()
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: nil) else {
        return false
    }

    try await context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: "Sign in to Kaya"
    )

    // Biometric passed — read refresh token from Keychain
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: "firebase_refresh_token",
        kSecReturnData as String: true,
    ]
    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    guard status == errSecSuccess, let data = result as? Data,
          let refreshToken = String(data: data, encoding: .utf8) else {
        return false
    }

    // Exchange for fresh ID token
    try await Auth.auth().signIn(withCustomToken: refreshToken)
    let idToken = try await Auth.auth().currentUser?.getIDToken()

    // Login with backend
    let resp = try await api.post("/auth/login", body: [
        "firebase_id_token": idToken,
        "device_id": deviceId,
    ])

    return resp["status"] == "ok"
}
```

**Important notes:**
- If the user changes their Face ID (adds a new face), iOS invalidates all Keychain items with `.biometryCurrentSet`. The app falls back to manual login.
- Biometric is per-device. Enabling on iPhone doesn't affect Android. Each device stores its own refresh token.
- After force-login from another device, the stored refresh token is revoked. Biometric unlock will succeed (local), but the Firebase refresh will fail. App should catch this and show manual login.

---

### 8. Change Password

```
POST /api/v1/auth/change-password
```

For phone+password users only. Requires current password.

**Headers:** Standard auth headers

**Body:**
```json
{
  "old_password": "OldPass123",
  "new_password": "NewPass456"
}
```

**Response (200):**
```json
{
  "status": "ok",
  "message": "Password changed."
}
```

**Errors:**
- `400` — Not a phone+password account / New password too short
- `401` — Incorrect current password

**Rate limit:** 5/minute

---

### 9. Reset Password (Forgot Password)

Forgot password works differently depending on how the user signed up.

---

#### 9a. Email + Password — Handled by Firebase (no backend call)

Firebase manages the email user's password entirely. The app calls Firebase directly — no backend endpoint is involved.

**Flow:**
1. User taps "Forgot Password" on the login screen
2. User enters their email address
3. App calls Firebase `sendPasswordResetEmail(email)`
4. Firebase sends a reset link to the user's inbox
5. User clicks the link, sets a new password on Firebase's hosted page
6. User returns to the app and logs in normally

**Code example (Flutter):**
```dart
await FirebaseAuth.instance.sendPasswordResetEmail(email: email);
```

**Code example (Swift):**
```swift
Auth.auth().sendPasswordReset(withEmail: email)
```

**Code example (React Native / JS):**
```js
await firebase.auth().sendPasswordResetEmail(email);
```

> No backend call needed. Firebase owns the email user's password completely.

---

#### 9b. Phone + Password — Requires Backend Call

```
POST /api/v1/auth/reset-password
```

For phone+password users who forgot their password. No auth header needed — the user proves identity via Firebase phone OTP.

**Flow:**
1. User taps "Forgot Password" on the login screen
2. User enters their phone number
3. App triggers Firebase phone OTP verification (`verifyPhoneNumber`)
4. User receives SMS code, enters it
5. Firebase verifies OTP, returns a fresh ID token
6. App calls this endpoint with that token + new password

**Body:**
```json
{
  "firebase_id_token": "eyJhbG...",
  "new_password": "NewPass456"
}
```

**Response (200):**
```json
{
  "status": "ok",
  "message": "Password has been reset."
}
```

**Errors:**
- `401` — Invalid or expired token
- `400` — Not a phone+password account / Password too short

**Rate limit:** 3/minute

---

#### Summary

| Auth Method | Forgot Password Flow | Backend Endpoint |
|---|---|---|
| Email + Password | Firebase `sendPasswordResetEmail()` — app only | Not needed |
| Phone + Password | Firebase OTP → `POST /auth/reset-password` | Required |
| Google / Apple | Firebase account recovery — app only | Not needed |

---

### 10. Setup Super Admin (One-Time)

```
POST /api/v1/auth/setup-super-admin
```

Promotes the calling user to super_admin. Only works if no super_admin exists in the system. One-time use.

**Headers:**
```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device-id>
```

**Response (200):**
```json
{
  "status": "ok",
  "user_id": "550e8400-...",
  "role": "super_admin",
  "message": "You are now the super admin."
}
```

**Errors:**
- `400` — Super admin already exists

---

### Auth Flow — Flutter/Dart Example

```dart
class KayaApi {
  final String baseUrl;
  String? _idToken;
  final String _deviceId = getDeviceId(); // Persist this

  // Login with Google
  Future<void> loginWithGoogle() async {
    // 1. Firebase client-side auth
    final googleUser = await GoogleSignIn().signIn();
    final googleAuth = await googleUser!.authentication;
    final credential = GoogleAuthProvider.credential(
      accessToken: googleAuth.accessToken,
      idToken: googleAuth.idToken,
    );
    final userCredential = await FirebaseAuth.instance.signInWithCredential(credential);
    _idToken = await userCredential.user!.getIdToken();

    // 2. Call Kaya backend
    final resp = await http.post(
      Uri.parse('$baseUrl/api/v1/auth/login'),
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        'firebase_id_token': _idToken,
        'device_id': _deviceId,
      }),
    );

    final data = jsonDecode(resp.body);
    if (data['status'] == 'device_conflict') {
      // Show dialog: "Active on another device. Switch?"
      // If yes → call /auth/force-login
    }
  }

  // Standard headers for all API calls
  Map<String, String> get headers => {
    'Content-Type': 'application/json',
    'Authorization': 'Bearer $_idToken',
    'X-Device-Id': _deviceId,
  };

  // 401 retry interceptor
  Future<http.Response> authenticatedRequest(
    String method, String path, {Map? body}
  ) async {
    var resp = await _send(method, path, body: body);

    if (resp.statusCode == 401) {
      // Token expired — refresh
      _idToken = await FirebaseAuth.instance.currentUser?.getIdToken(true);
      resp = await _send(method, path, body: body);
    }

    return resp;
  }
}
```

### Auth Flow — Swift/iOS Example

```swift
class KayaAPI {
    let baseURL: String
    let deviceId: String // UIDevice.current.identifierForVendor?.uuidString

    // Login with Google/Apple
    func login(firebaseToken: String) async throws -> LoginResponse {
        let body: [String: Any] = [
            "firebase_id_token": firebaseToken,
            "device_id": deviceId
        ]
        return try await post("/api/v1/auth/login", body: body)
    }

    // Standard headers
    func authHeaders() async -> [String: String] {
        let token = try? await Auth.auth().currentUser?.getIDToken()
        return [
            "Authorization": "Bearer \(token ?? "")",
            "X-Device-Id": deviceId,
            "Content-Type": "application/json"
        ]
    }

    // 401 retry
    func authenticatedRequest(_ request: URLRequest) async throws -> Data {
        var resp = try await URLSession.shared.data(for: request)
        if (resp.1 as? HTTPURLResponse)?.statusCode == 401 {
            let freshToken = try await Auth.auth().currentUser?.getIDTokenForcingRefresh(true)
            // Update Authorization header and retry
            var retryRequest = request
            retryRequest.setValue("Bearer \(freshToken ?? "")", forHTTPHeaderField: "Authorization")
            resp = try await URLSession.shared.data(for: retryRequest)
        }
        return resp.0
    }
}
```

---

## Coaching Sessions

### 1. Start a Session

```
POST /api/v1/sessions/start
```

**Body:**
```json
{
  "type": "intro"
}
```

`type` must be one of: `intro`, `foundation`, `followup`

**Response (200):**
```json
{
  "session_id": "uuid-here",
  "type": "intro",
  "status": "active",
  "time_remaining_seconds": 86400,
  "expires_at": "2026-03-10T11:23:59.320390+00:00"
}
```

**Errors:**
- `400` — Already have an active session / Invalid type
- `402` — No credits (paid sessions only)
- `400` — Journey not ready (e.g. trying followup before completing foundation)

**Session journey order:**
1. `intro` — Always free. No prerequisites. Unlimited repeats.
2. `foundation` — First one is free, rest cost 1 credit. Requires completed intro.
3. `followup` — Always costs 1 credit. Requires completed foundation.

---

### 2. Send Message (Streaming)

> This is the **primary** message endpoint. Tokens arrive in real-time so the user sees text appearing word by word.

```
POST /api/v1/sessions/{session_id}/message/stream
```

**Body:**
```json
{
  "message": "Hi, I want to improve my sleep"
}
```

**Response:** `text/event-stream` (SSE)

#### SSE Event Types:

**1. Status — LLM is thinking**
```
data: {"event": "status", "data": "thinking"}

```

**2. Status — Tool is being used**
```
data: {"event": "status", "data": "tool_use", "tool": "update_user_memory"}

```

**3. Token — Streamed text (word by word)**
```
data: {"event": "token", "data": "I'd "}

data: {"event": "token", "data": "love "}

data: {"event": "token", "data": "to help!"}

```

**4. Done — Final complete response (text)**
```
data: {"event": "done", "message_type": "text", "response": "I'd love to help!"}

```

**5. Done — Final response with options/buttons**
```
data: {"event": "done", "message_type": "options", "response": "What's your main concern?", "options": ["Sleep", "Energy", "Digestion"]}

```

**6. Error**
```
data: {"event": "error", "data": "I apologize for the inconvenience. Please try again."}

```

#### How the app should handle events:

1. On `status` → Show "thinking..." or "searching..." indicator
2. On `token` → Append `data` to the chat bubble in real-time
3. On `done` → Use `response` as the final complete message. If `message_type` is `"options"`, display `options` as tappable buttons. User taps a button → send that text as the next message.
4. On `error` → Show error message to user

#### Flutter/Dart example:
```dart
final request = http.Request('POST',
  Uri.parse('$baseUrl/api/v1/sessions/$sessionId/message/stream'));
request.headers.addAll(api.headers);
request.body = jsonEncode({'message': userMessage});

final response = await http.Client().send(request);
String fullText = '';

response.stream
  .transform(utf8.decoder)
  .transform(const LineSplitter())
  .listen((line) {
    if (line.startsWith('data: ')) {
      final json = jsonDecode(line.substring(6));

      switch (json['event']) {
        case 'status':
          showThinkingIndicator();
          break;
        case 'token':
          fullText += json['data'];
          updateChatBubble(fullText);
          break;
        case 'done':
          final response = json['response'];
          if (json['message_type'] == 'options') {
            showOptionButtons(json['options']);
          }
          break;
        case 'error':
          showError(json['data']);
          break;
      }
    }
  });
```

#### Swift/iOS example:
```swift
var request = URLRequest(url: URL(string: "\(baseURL)/api/v1/sessions/\(sessionId)/message/stream")!)
request.httpMethod = "POST"
request.allHTTPHeaderFields = await authHeaders()
request.httpBody = try? JSONEncoder().encode(["message": userMessage])

let (bytes, _) = try await URLSession.shared.bytes(for: request)
var fullText = ""

for try await line in bytes.lines {
    guard line.hasPrefix("data: ") else { continue }
    let jsonStr = String(line.dropFirst(6))
    guard let data = jsonStr.data(using: .utf8),
          let json = try? JSONSerialization.jsonObject(with: data) as? [String: Any],
          let event = json["event"] as? String else { continue }

    switch event {
    case "status":
        showThinkingIndicator()
    case "token":
        fullText += json["data"] as? String ?? ""
        updateChatBubble(fullText)
    case "done":
        if json["message_type"] as? String == "options" {
            showOptionButtons(json["options"] as? [String] ?? [])
        }
    case "error":
        showError(json["data"] as? String ?? "Unknown error")
    default:
        break
    }
}
```

---

### 3. Send Message (Non-Streaming)

```
POST /api/v1/sessions/{session_id}/message
```

Same body as streaming. Returns the full response at once (slower — 10-15 seconds wait). Use streaming instead.

**Response (200):**
```json
{
  "response": "I'd love to help you improve your sleep!",
  "message_type": "text"
}
```

Or with options:
```json
{
  "response": "What's your main concern?",
  "message_type": "options",
  "options": ["Sleep", "Energy", "Digestion"]
}
```

---

### 4. Get User Stats (Home Screen)

```
GET /api/v1/sessions/stats/me
```

**Response (200):**
```json
{
  "sessions_completed": {
    "intro": 1,
    "foundation": 2,
    "followup": 3
  },
  "sessions_total": {
    "intro": 1,
    "foundation": 3,
    "followup": 4
  },
  "credits": {
    "paid_balance": 3,
    "intro_sessions_done": 1,
    "foundation_sessions_done": 3
  },
  "journey_stage": "followup"
}
```

`journey_stage` — where the user is: `"intro"`, `"foundation"`, or `"followup"`.

---

### 5. Get Active Session

```
GET /api/v1/sessions/active
```

**Response (200):**
```json
{
  "active_session": {
    "session_id": "uuid-here",
    "type": "intro",
    "status": "active",
    "time_remaining_seconds": 85200,
    "progress": {"welcomed_user": true, "explained_24h_timer": false},
    "started_at": "2026-03-09T10:00:00+00:00",
    "expires_at": "2026-03-10T10:00:00+00:00"
  }
}
```

Or if no active session:
```json
{
  "active_session": null
}
```

---

### 6. Get Session Messages (View-Only History)

```
GET /api/v1/sessions/{session_id}/messages
```

Works for any session — active, completed, or expired. Read-only.

Only real conversation messages are returned. Internal tool exchange rows (memory lookups, goal saves, etc.) are filtered out automatically.

**Response (200):**
```json
{
  "session_id": "uuid-here",
  "type": "intro",
  "status": "completed",
  "messages": [
    {
      "role": "user",
      "content": "Hello, I want to improve my health",
      "message_type": "text",
      "created_at": "2026-03-09T10:01:00+00:00"
    },
    {
      "role": "assistant",
      "content": "Welcome! I'm Kaya, your AI health coach...",
      "message_type": "text",
      "created_at": "2026-03-09T10:01:05+00:00"
    },
    {
      "role": "assistant",
      "content": "Which area would you like to focus on first?{\"text\":\"Which area?\",\"options\":[\"Sleep\",\"Stress\",\"Nutrition\"]}",
      "message_type": "options",
      "created_at": "2026-03-09T10:02:00+00:00"
    },
    {
      "role": "assistant",
      "content": "Great session! You're ready to start your Foundation session.",
      "message_type": "session_ended",
      "created_at": "2026-03-09T10:45:00+00:00"
    }
  ]
}
```

**`message_type` values in history**

| `message_type` | What it means | How to render |
|---|---|---|
| `text` | Normal AI message | Plain chat bubble |
| `options` | AI offered button choices | Chat bubble — re-render buttons if desired (tapping re-sends that text) |
| `session_ended` | AI ended the session | Final bubble — show "Start next session" prompt |
| `session_suggestion` | AI suggested starting a session (chat only) | Bubble + "Start Session" button |

> See `API_DOCS_CHAT_SESSION.md` for the full breakdown of each `message_type`, including response shapes, Flutter examples, and rendering guidance.

---

### 7. Get Session Detail

```
GET /api/v1/sessions/{session_id}
```

**Response (200):**
```json
{
  "session_id": "uuid-here",
  "type": "intro",
  "status": "active",
  "progress": {"welcomed_user": true},
  "started_at": "2026-03-09T10:00:00+00:00",
  "completed_at": null,
  "completion_reason": null
}
```

---

### 8. List All Sessions

```
GET /api/v1/sessions
```

Returns coaching sessions only (excludes general chat threads).

**Response (200):**
```json
{
  "sessions": [
    {
      "session_id": "uuid",
      "type": "intro",
      "status": "completed",
      "started_at": "2026-03-09T10:00:00+00:00",
      "completed_at": "2026-03-09T11:00:00+00:00"
    }
  ]
}
```

---

## General Chat (Non-Coaching)

Free, unlimited. No timer. No credits needed.

### 1. Create a Chat Thread

```
POST /api/v1/chat/threads
```

**Response (200):**
```json
{
  "thread_id": "uuid-here",
  "type": "general",
  "status": "active",
  "created_at": "2026-03-09T10:00:00+00:00"
}
```

---

### 2. Send Chat Message (Streaming)

Same SSE format as coaching sessions.

```
POST /api/v1/chat/threads/{thread_id}/message/stream
```

**Body:**
```json
{
  "message": "What foods help with inflammation?"
}
```

**Response:** `text/event-stream` — same event types as coaching sessions.

---

### 3. Send Chat Message (Non-Streaming)

```
POST /api/v1/chat/threads/{thread_id}/message
```

Same body, returns full response at once.

---

### 4. List Chat Threads

```
GET /api/v1/chat/threads
```

**Response (200):**
```json
{
  "threads": [
    {
      "thread_id": "uuid",
      "title": "What foods help with...",
      "created_at": "2026-03-09T10:00:00+00:00",
      "updated_at": "2026-03-09T10:05:00+00:00"
    }
  ]
}
```

---

### 5. Get Thread Detail

```
GET /api/v1/chat/threads/{thread_id}
```

**Response (200):**
```json
{
  "thread_id": "uuid",
  "title": "What foods help with...",
  "messages": [
    {
      "id": "uuid",
      "role": "user",
      "content": "What foods help with inflammation?",
      "message_type": "text",
      "created_at": "2026-03-09T10:00:00+00:00"
    },
    {
      "id": "uuid",
      "role": "assistant",
      "content": "Great question! Omega-3 rich foods...",
      "message_type": "text",
      "created_at": "2026-03-09T10:00:05+00:00"
    }
  ]
}
```

---

### 6. Delete Thread

```
DELETE /api/v1/chat/threads/{thread_id}
```

**Response (200):**
```json
{
  "status": "deleted"
}
```

---

## User Data & Settings (`/me`)

All `/me` endpoints require standard auth headers.

### 1. Get Account

```
GET /api/v1/me/account
```

**Response (200):**
```json
{
  "user_id": "550e8400-...",
  "name": "Sarah",
  "email": "sarah@example.com",
  "phone": null,
  "auth_provider": "google",
  "biometric_enabled": false,
  "data_sharing_consent": false,
  "otp_verified": true,
  "created_at": "2026-03-09T10:00:00+00:00"
}
```

---

### 2. Update Account

```
PUT /api/v1/me/account
```

Currently only `name` is editable (email/phone come from Firebase).

**Body:**
```json
{
  "name": "Sarah H."
}
```

**Response (200):**
```json
{
  "status": "ok",
  "name": "Sarah H."
}
```

Also syncs the name to the AI coach's memory — the coach will use the updated name in future conversations.

**Errors:**
- `422` — Empty name or exceeds 100 chars

---

### 3. Get Consent

```
GET /api/v1/me/consent
```

**Response (200):**
```json
{
  "data_sharing_consent": false
}
```

---

### 4. Update Consent

```
PUT /api/v1/me/consent
```

**Body:**
```json
{
  "data_sharing_consent": true
}
```

**Response (200):**
```json
{
  "data_sharing_consent": true
}
```

---

### 5. Submit Feedback

```
POST /api/v1/me/feedback
```

**Body:**
```json
{
  "category": "idea",
  "message": "Add dark mode please!"
}
```

`category` must be one of: `idea`, `complaint`, `general`

**Response (200):**
```json
{
  "status": "ok",
  "feedback_id": "uuid-here"
}
```

**Errors:**
- `422` — Invalid category / Empty message / Message over 2000 chars

---

### 6. List Goals

```
GET /api/v1/me/goals
GET /api/v1/me/goals?status=active
```

Optional filter: `status` — `active`, `completed`, or `paused`

**Response (200):**
```json
{
  "goals": [
    {
      "id": "uuid",
      "title": "Improve sleep quality",
      "description": "Focus on getting 7-8 hours of restorative sleep",
      "pillar": "sleep",
      "timeframe": "3 months",
      "status": "active",
      "smart": {
        "specific": "Go to bed by 10:30pm every night",
        "measurable": "Track with sleep diary",
        "achievable": "Start with 3 nights per week",
        "relevant": "Better sleep improves energy and mood",
        "timebound": "Achieve consistent schedule in 3 months"
      },
      "created_at": "2026-03-10T11:00:00+00:00",
      "updated_at": "2026-03-10T11:00:00+00:00"
    }
  ],
  "count": 1
}
```

Fields like `description`, `pillar`, `timeframe`, and all `smart.*` fields can be `null` (set by the AI during coaching).

---

### 7. Get Goal Detail

```
GET /api/v1/me/goals/{goal_id}
```

Same fields as list, plus `habits` — habits linked to this goal.

**Response (200):**
```json
{
  "id": "uuid",
  "title": "Improve sleep quality",
  "status": "active",
  "smart": { ... },
  "habits": [
    {
      "id": "uuid",
      "name": "No screens after 9pm",
      "frequency": "daily",
      "current_streak": 5,
      "best_streak": 12,
      "is_active": true
    }
  ]
}
```

---

### 8. List Habits

```
GET /api/v1/me/habits
```

Returns active habits only.

**Response (200):**
```json
{
  "habits": [
    {
      "id": "uuid",
      "name": "Morning meditation",
      "frequency": "daily",
      "goal_id": "uuid-or-null",
      "reminder_time": "07:00:00",
      "current_streak": 3,
      "best_streak": 10,
      "created_at": "2026-03-10T11:00:00+00:00"
    }
  ],
  "count": 1
}
```

---

### 9. Today's Habits

```
GET /api/v1/me/habits/today
```

Habits with today's completion status. For a daily checklist UI.

**Response (200):**
```json
{
  "date": "2026-03-12",
  "habits": [
    {
      "id": "uuid",
      "name": "Morning meditation",
      "frequency": "daily",
      "current_streak": 3,
      "best_streak": 10,
      "completed_today": false
    }
  ],
  "total": 2,
  "completed": 0
}
```

---

### 10. Log Habit (Mark Done/Undone)

```
POST /api/v1/me/habits/{habit_id}/log
```

**Body (optional):**
```json
{
  "completed": true
}
```

`completed` defaults to `true`. Set to `false` to undo.

**Response (200):**
```json
{
  "status": "logged",
  "date": "2026-03-12",
  "streak": 4
}
```

Other statuses: `already_logged`, `unlogged`, `not_logged`. `streak` only present for `logged` and `unlogged`.

---

### 11. Mood History

```
GET /api/v1/me/mood
GET /api/v1/me/mood?limit=7
```

**Response (200):**
```json
{
  "entries": [
    {"rating": 7, "note": "Feeling good after morning walk", "date": "2026-03-12"},
    {"rating": 5, "note": null, "date": "2026-03-11"}
  ],
  "latest": 7,
  "previous": 5,
  "average": 6.0,
  "trend": "up",
  "count": 2
}
```

`trend` — `"up"`, `"down"`, `"stable"`, or `null`. Mood is recorded by the AI during sessions (1-10 scale).

---

### 12. Profile (AI Memory)

```
GET /api/v1/me/profile
```

What Kaya remembers about the user — key-value pairs saved by the AI during coaching.

**Response (200):**
```json
{
  "memory": {
    "name": {
      "value": "Sarah",
      "source": "intro_session",
      "updated_at": "2026-03-10T11:00:00+00:00"
    },
    "health_focus": {
      "value": "improving energy and reducing brain fog",
      "source": "foundation_session",
      "updated_at": "2026-03-10T12:00:00+00:00"
    }
  },
  "fields": 2
}
```

Keys are **dynamic** — the AI decides what to remember. Common keys: `name`, `health_focus`, `occupation`, `values`, `barriers`, `support_system`.

---

### 13. Check-in Schedule

```
GET /api/v1/me/checkins
```

**Response (200):**
```json
{
  "schedule": {
    "frequency_days": 2,
    "is_active": true,
    "last_sent_at": "2026-03-10T08:00:00+00:00",
    "next_due_at": "2026-03-12T08:00:00+00:00"
  }
}
```

Or `{"schedule": null}` if not set (set by the AI during foundation sessions).

---

## Admin Endpoints

> Requires `admin` or `super_admin` role. All admin endpoints require standard auth headers.

### 1. List All Sessions

```
GET /api/v1/admin/sessions
GET /api/v1/admin/sessions?user_id=uuid&session_type=intro&status=active&limit=50&offset=0
```

### 2. Session Detail

```
GET /api/v1/admin/sessions/{session_id}
```

Returns full session with all messages, tool calls, user memory snapshot.

### 3. List All Users

```
GET /api/v1/admin/users
```

Returns all users with session counts, goal counts, token usage.

### 4. User Detail

```
GET /api/v1/admin/users/{user_id}
```

Full user data — memory, goals, habits, moods, sessions.

### 5. Reset Test User

```
POST /api/v1/admin/users/{user_id}/reset
```

Deletes all data for a user and resets credits. For testing.

### 6. Add Credits

```
POST /api/v1/admin/users/{user_id}/credits
Body: {"amount": 5}
```

### 7. Change User Role (Super Admin Only)

```
PUT /api/v1/admin/users/{user_id}/role
Body: {"role": "admin"}
```

`role` — one of: `user`, `admin`, `super_admin`

### 8. Suspend / Unsuspend User

```
PUT /api/v1/admin/users/{user_id}/suspend
PUT /api/v1/admin/users/{user_id}/unsuspend
```

### 9. Admin Profile

```
GET /api/v1/admin/me
```

### 10. Debug Endpoints

```
GET /api/v1/admin/debug/timing?query=...&session_type=intro
GET /api/v1/admin/debug/rag?query=...&namespace=fm_knowledge
GET /api/v1/admin/debug/llm?prompt_size=full&tools=true
```

---

## Health Check

```
GET /health
```

Public. No auth required.

```json
{
  "status": "ok"
}
```

---

## Error Responses

All errors follow this format:

```json
{
  "detail": "Error message here"
}
```

| Status Code | Meaning |
|-------------|---------|
| 400 | Bad request (invalid type, session expired, journey not ready) |
| 401 | Invalid/expired/revoked token |
| 402 | No credits — need to purchase |
| 403 | Not your session / Wrong device / Account suspended / Insufficient role |
| 404 | Not found |
| 422 | Validation error (missing fields, invalid format) |
| 429 | Rate limited — response body is **plain text**, not JSON. Always check `statusCode == 429` before calling `jsonDecode`. |
| 500 | Server error |

---

## Typical App Flow

```
First Launch:
1. User signs up (Google/Apple/Email/Phone) via Firebase client SDK
2. POST /auth/signup → register with backend
3. GET  /sessions/stats/me → home screen data (journey_stage: "intro")
4. POST /sessions/start {type: "intro"} → start first session
5. POST /sessions/{id}/message/stream → chat with AI coach

Returning User:
1. Firebase SDK auto-refreshes token
2. POST /auth/login → check device, get user info
3. GET  /sessions/active → resume active session if any
4. GET  /sessions/stats/me → home screen data

General Chat:
1. GET  /chat/threads → list past conversations
2. POST /chat/threads → create new thread
3. POST /chat/threads/{id}/message/stream → chat

Settings:
1. GET  /me/account → show profile
2. PUT  /me/account → update name
3. POST /auth/enable-biometric → enable biometric
4. POST /auth/change-password → change password
5. GET  /me/consent → show consent toggle
6. PUT  /me/consent → update consent
7. POST /me/feedback → submit feedback

Dashboard / Progress:
1. GET  /me/goals → list goals
2. GET  /me/habits/today → daily checklist
3. POST /me/habits/{id}/log → mark habit done
4. GET  /me/mood → mood history + trend
5. GET  /me/profile → what Kaya remembers
```

---

## Debug Mode

Add `?debug=true` to any message endpoint to include RAG retrieval, tool usage, and pattern detection in the response. Not for end users.

```
POST /api/v1/sessions/{id}/message?debug=true
POST /api/v1/sessions/{id}/message/stream?debug=true
POST /api/v1/chat/threads/{id}/message?debug=true
POST /api/v1/chat/threads/{id}/message/stream?debug=true
```

---

## Important Notes

1. **Always use `/stream` endpoints for messages.** Non-streaming takes 10-15 seconds. Streaming shows text within ~3 seconds.

2. **Check for active session** before starting a new one. `GET /sessions/active` first.

3. **Session timer**: Coaching sessions expire after 24 hours. Use `time_remaining_seconds` to show countdown.

4. **Options/buttons**: When `message_type` is `"options"`, display `options` as tappable buttons.

5. **Credits**: Intro = always free. First foundation = free. Everything else = 1 credit. `402` = no credits.

6. **General chat** is free and unlimited. No timer. No credits.

7. **Token refresh**: Implement the 401 retry interceptor. Tokens expire every 60 minutes.

8. **Device ID**: Must be persistent across app launches. Use the device's hardware ID.
