# Phone Auth Bug Fixes

Three bugs found that together cause Firebase error code 39 (`TOO_MANY_REQUESTS`) on the
second registration attempt, leaving the phone number blocked for ~1 hour.

---

## Bug 1 — Stale `_resendToken` passed for a different phone number

**File:** `auth_service.dart`

**Root cause:**  
`_verificationId` and `_resendToken` are set during the first registration and stored as
instance variables on the `AuthService` singleton. The `signOut()` method never clears them.
On the next registration (different phone number), `forceResendingToken: _resendToken` passes
the old token — which belongs to the previous number — to Firebase's `verifyPhoneNumber`.

Firebase treats a resend token issued for number A being used for number B as a suspicious
pattern and immediately returns `TOO_MANY_REQUESTS` (code 39) **before sending any SMS**
(which is why no Firebase credits are consumed). Each retry deepens the rate-limit ban.

---

### Fix 1a — Clear tokens on `signOut()`

**Location:** `auth_service.dart` around line 1738

**Before:**
```dart
_idToken = null;
authStatus.value = AuthStatus.unauthenticated;
biometricEnabled.value = false;
```

**After:**
```dart
_idToken = null;
_verificationId = null;   // clear stale phone verification session
_resendToken = null;      // clear stale resend token (bound to previous number)
authStatus.value = AuthStatus.unauthenticated;
biometricEnabled.value = false;
```

---

### Fix 1b — Only pass `forceResendingToken` when resending to the same number

**Location:** `auth_service.dart` around line 638 and line 642

Add a field to track the last verified number, then only pass the resend token when the
number matches.

**Before (fields):**
```dart
String? _verificationId;
int? _resendToken;
```

**After (fields):**
```dart
String? _verificationId;
int? _resendToken;
String? _lastVerifiedPhone;   // track which number the resend token belongs to
```

**Before (`startPhoneVerification` signature + body start):**
```dart
Future<void> startPhoneVerification({
  required String phoneNumber,
  required Function(String verificationId) onCodeSent,
  required Function(String error) onError,
  Function(PhoneAuthCredential credential)? onAutoVerified,
}) async {
  AppLogger.info(
    _tag,
    'startPhoneVerification',
    'Starting verification for: $phoneNumber',
  );
  isLoading.value = true;

  await _firebaseAuth.verifyPhoneNumber(
    phoneNumber: phoneNumber,
    // ... callbacks ...
    forceResendingToken: _resendToken,
  );
```

**After:**
```dart
Future<void> startPhoneVerification({
  required String phoneNumber,
  required Function(String verificationId) onCodeSent,
  required Function(String error) onError,
  Function(PhoneAuthCredential credential)? onAutoVerified,
}) async {
  AppLogger.info(
    _tag,
    'startPhoneVerification',
    'Starting verification for: $phoneNumber',
  );
  isLoading.value = true;

  // Only pass resend token if it belongs to THIS number.
  // Passing a token for a different number causes Firebase to return
  // TOO_MANY_REQUESTS (code 39) before any SMS is sent.
  final resendToken =
      (_lastVerifiedPhone == phoneNumber) ? _resendToken : null;

  await _firebaseAuth.verifyPhoneNumber(
    phoneNumber: phoneNumber,
    // ... callbacks ...
    forceResendingToken: resendToken,
  );
```

Also update the `codeSent` callback to record which number the token belongs to:

**Before (`codeSent` callback):**
```dart
codeSent: (verificationId, resendToken) {
  AppLogger.success(_tag, 'startPhoneVerification', 'OTP code sent');
  isLoading.value = false;
  _verificationId = verificationId;
  _resendToken = resendToken;
  onCodeSent(verificationId);
},
```

**After:**
```dart
codeSent: (verificationId, resendToken) {
  AppLogger.success(_tag, 'startPhoneVerification', 'OTP code sent');
  isLoading.value = false;
  _verificationId = verificationId;
  _resendToken = resendToken;
  _lastVerifiedPhone = phoneNumber;   // record which number this token is for
  onCodeSent(verificationId);
},
```

And clear `_lastVerifiedPhone` in `signOut()` alongside the other fields:

```dart
_idToken = null;
_verificationId = null;
_resendToken = null;
_lastVerifiedPhone = null;   // clear alongside resend token
authStatus.value = AuthStatus.unauthenticated;
biometricEnabled.value = false;
```

---

## Bug 2 — Silent country code guessing instead of user warning

**Files:** `auth_controller.dart`, `create_account_view.dart`

**Root cause:**  
`_formatToE164` silently auto-formats any number that doesn't start with `+` by guessing
a default country code. When the guess is wrong, Firebase sends the OTP to a completely
different number (or rejects it server-side). The user sees nothing, retries, and burns
through the Firebase rate limit quota — all without knowing their input was invalid.

Silently guessing is the wrong approach regardless of which country code is used as default.
The correct approach is to **require the user to include the country code** and show a clear
warning if they don't.

---

### Fix 2a — Replace silent formatting with a validation warning in `auth_controller.dart`

**Location:** `auth_controller.dart` around line 172 (`startPhoneVerification`) and line 527 (`_formatToE164`)

The `_formatToE164` method should only strip formatting characters and validate — it should
not guess or inject a country code. Add an explicit check before calling it: if the number
doesn't start with `+`, stop immediately and show a warning.

**Before (`startPhoneVerification` in `auth_controller.dart ~172`):**
```dart
// Format phone number to E.164 format
String formattedPhone = _formatToE164(phone);

// Validate E.164 format
if (!_isValidE164(formattedPhone)) {
  errorMessage.value = 'Invalid phone format. Use: +628123456789';
  return;
}
```

**After:**
```dart
// Require the user to provide the country code explicitly.
// Never silently guess — a wrong guess sends OTP to a foreign number
// with no feedback, causing retries that exhaust Firebase rate limits.
if (!phone.trimLeft().startsWith('+')) {
  errorMessage.value = 'Please include your country code, e.g. +962 7X XXX XXXX';
  return;
}

// Strip formatting characters only — no country code injection
String formattedPhone = _formatToE164(phone);

// Validate E.164 format
if (!_isValidE164(formattedPhone)) {
  errorMessage.value = 'Invalid phone format. Use: +962 7X XXX XXXX';
  return;
}
```

Apply the same change to `signInWithPhone` (`auth_controller.dart ~283`):

**Before:**
```dart
// Format phone number to E.164
String formattedPhone = _formatToE164(phone);

if (!_isValidE164(formattedPhone)) {
  errorMessage.value = 'Invalid phone format. Use: +628123456789';
  return;
}
```

**After:**
```dart
if (!phone.trimLeft().startsWith('+')) {
  errorMessage.value = 'Please include your country code, e.g. +962 7X XXX XXXX';
  return;
}

String formattedPhone = _formatToE164(phone);

if (!_isValidE164(formattedPhone)) {
  errorMessage.value = 'Invalid phone format. Use: +962 7X XXX XXXX';
  return;
}
```

**Replace `_formatToE164` entirely** — it no longer needs to guess:

**Before:**
```dart
String _formatToE164(String phone) {
  String cleaned = phone.replaceAll(RegExp(r'[^0-9+]'), '');
  if (cleaned.startsWith('+')) {
    return cleaned;
  }
  cleaned = cleaned.replaceFirst(RegExp(r'^0+'), '');
  if (cleaned.startsWith('62')) {
    return '+$cleaned';
  }
  // Default: add Indonesia country code (+62)
  return '+62$cleaned';
}
```

**After:**
```dart
/// Strips formatting characters only. Caller must validate that
/// the input already starts with '+' before calling this.
String _formatToE164(String phone) {
  // Remove spaces, dashes, parentheses — keep digits and leading +
  return phone.replaceAll(RegExp(r'[^\d+]'), '');
}
```

---

### Fix 2b — Update hint text in `create_account_view.dart`

**Location:** `create_account_view.dart:186`

Make the hint text explicitly show the `+` is required so users know what format to use.

**Before:**
```dart
hintText: '+(962) 7X XXX-XXXX',
```

**After:**
```dart
hintText: '+962 7X XXX XXXX  (include + and country code)',
```

---

## Summary

| # | File | Line | Change |
|---|------|------|--------|
| 1a | `auth_service.dart` | ~1738 | Add `_verificationId = null` and `_resendToken = null` in `signOut()` |
| 1b | `auth_service.dart` | ~638 | Add `String? _lastVerifiedPhone` field |
| 1b | `auth_service.dart` | ~653 | Compute `resendToken` conditionally before `verifyPhoneNumber` |
| 1b | `auth_service.dart` | ~689 | Set `_lastVerifiedPhone = phoneNumber` in `codeSent` callback |
| 1b | `auth_service.dart` | ~1738 | Also clear `_lastVerifiedPhone = null` in `signOut()` |
| 2a | `auth_controller.dart` | ~172 | Add `+` prefix check before `startPhoneVerification`, show warning if missing |
| 2a | `auth_controller.dart` | ~283 | Same `+` prefix check in `signInWithPhone` |
| 2a | `auth_controller.dart` | ~527–546 | Replace `_formatToE164` — remove country code guessing logic entirely |
| 2b | `create_account_view.dart` | ~186 | Update hint text to make `+` and country code visually required |

---

## Current state workaround

The phone number that triggered the "too many requests" block is locked on Firebase's
servers for approximately **1 hour**. There is no client-side workaround — uninstalling the
app does not help. Wait 1 hour and the number will be unblocked automatically.
