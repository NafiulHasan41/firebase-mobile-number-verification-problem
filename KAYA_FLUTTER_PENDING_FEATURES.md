# Kaya — Flutter Pending Features Guide

Four features that still need to be implemented on the Flutter client side.
Priority order: **Delete Account → Analytics Events → Voice Input → Profile Picture Upload**

---

## Base URL

| Environment | URL |
|---|---|
| Test server | `https://test.globalvin.co/kayatest` |
| Production | TBD |

## Common Headers (all authenticated endpoints)

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <persistent_device_uuid>
Content-Type: application/json
```

### X-Device-Id — Required on every authenticated request

This is **not** a testing artefact — it is a permanent production requirement. The backend uses it to enforce single-device sessions. If the same account logs in on a second device, all requests from the first device are rejected with 403.

**Generate once on first install, never change it:**

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:uuid/uuid.dart';

final _storage = FlutterSecureStorage();

Future<String> getDeviceId() async {
  String? id = await _storage.read(key: 'device_id');
  if (id == null) {
    id = const Uuid().v4();
    await _storage.write(key: 'device_id', value: id);
  }
  return id;
}
```

Rules:
- Generate once → store in `SecureStorage` → reuse forever
- Must survive app restarts and updates
- Only regenerate if the app is fully uninstalled and reinstalled

---

## 1. Account Deletion (Required — App Store / GDPR)

Apple App Store **requires** in-app account deletion. Without this the iOS build will be rejected.

### Endpoint

```
DELETE /api/v1/me/account
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_id>
Content-Type: application/json
```

### Request Body

```json
{
  "confirm": true
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `confirm` | boolean | Yes | Must be `true` to proceed. Prevents accidental deletion. |

---

### Success Response

**HTTP 204 No Content**

No response body. Empty response means the account was fully deleted.

```
HTTP/1.1 204 No Content
```

What happened on the backend:
- Active PayPal subscription cancelled
- All user data deleted (habits, goals, moods, balance, payments, notifications, memory, device tokens)
- User row anonymised (name, email, phone → NULL)
- Firebase account deleted — the token used in this request is now invalid

---

### Error Responses

**HTTP 400 — Confirm not provided**
```json
{
  "detail": "Set confirm=true to permanently delete your account"
}
```
Cause: `confirm` field is `false` or missing. Show user a proper confirmation dialog before calling.

---

**HTTP 401 — Not authenticated**
```json
{
  "detail": "Missing Bearer token"
}
```
```json
{
  "detail": "Token expired. Please refresh."
}
```
```json
{
  "detail": "Invalid token."
}
```
Cause: No `Authorization` header, expired or invalid Firebase token. Call `FirebaseAuth.instance.currentUser?.getIdToken(true)` to force-refresh the token and retry once. If still 401, redirect to login screen.

---

**HTTP 403 — Session active on another device**
```json
{
  "detail": "Session active on another device. Use /auth/force-login to switch."
}
```
Cause: The same Firebase account is currently logged in on a different device. `X-Device-Id` is a **permanent, required header** on every authenticated request — it is not just for testing. Generate it once on install, persist in `SecureStorage`, send on every call. To switch devices, call `POST /api/v1/auth/force-login`.

---

**HTTP 404 — Account not found**
```json
{
  "detail": "Account not found"
}
```
Cause: User is already deleted or the user ID does not exist. Clear local session and navigate to onboarding.

---

**HTTP 500 — Server error**
```json
{
  "detail": "Internal Server Error"
}
```
Cause: Unexpected server error. Show "Something went wrong, please try again." Do **not** sign the user out — their account may still exist.

---

### Flutter Implementation

**Where to put it:** Profile → Settings → Delete Account

**Recommended UX flow:**
1. Show confirmation dialog:
   - Title: "Delete your account?"
   - Body: "This will permanently delete your account and all your data. This cannot be undone."
   - Buttons: "Cancel" | "Delete Account" (red/destructive)
2. On confirm → show loading indicator → call API
3. On `204` → clear local storage → sign out of Firebase → navigate to onboarding
4. On any error → dismiss loading → show error message → stay on screen

```dart
Future<void> deleteAccount(BuildContext context) async {
  // Step 1: Confirm dialog
  final confirmed = await showDialog<bool>(
    context: context,
    builder: (_) => AlertDialog(
      title: const Text('Delete your account?'),
      content: const Text(
        'This will permanently delete your account and all your data. This cannot be undone.',
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: const Text('Cancel'),
        ),
        TextButton(
          onPressed: () => Navigator.pop(context, true),
          style: TextButton.styleFrom(foregroundColor: Colors.red),
          child: const Text('Delete Account'),
        ),
      ],
    ),
  );

  if (confirmed != true) return;

  // Step 2: Call API
  try {
    final token = await FirebaseAuth.instance.currentUser?.getIdToken(true);
    final response = await http.delete(
      Uri.parse('$baseUrl/api/v1/me/account'),
      headers: {
        'Authorization': 'Bearer $token',
        'X-Device-Id': deviceId,
        'Content-Type': 'application/json',
      },
      body: jsonEncode({'confirm': true}),
    );

    // Step 3: Handle response
    if (response.statusCode == 204) {
      // Success — clear everything and go to onboarding
      await _clearLocalData();
      await FirebaseAuth.instance.signOut();
      Navigator.pushNamedAndRemoveUntil(context, '/onboarding', (_) => false);

    } else if (response.statusCode == 401) {
      _showError(context, 'Session expired. Please log in again.');

    } else if (response.statusCode == 404) {
      // Already deleted — clean up anyway
      await _clearLocalData();
      await FirebaseAuth.instance.signOut();
      Navigator.pushNamedAndRemoveUntil(context, '/onboarding', (_) => false);

    } else {
      _showError(context, 'Something went wrong. Please try again.');
    }

  } catch (e) {
    _showError(context, 'No internet connection. Please try again.');
  }
}

Future<void> _clearLocalData() async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.clear();
  // Also clear any SecureStorage, cached files, etc.
}

void _showError(BuildContext context, String message) {
  ScaffoldMessenger.of(context).showSnackBar(
    SnackBar(content: Text(message), backgroundColor: Colors.red),
  );
}
```

---

## 2. Analytics Events (Required — Funnel Tracking)

The backend tracks everything after signup. Flutter must fire events for the top-of-funnel steps the backend cannot see.

### Endpoint

```
POST /api/v1/events
Content-Type: application/json
```

**Auth: Optional** — works with or without a Bearer token. Always send `anonymous_id`.

### Request Body

```json
{
  "event_type": "app_install",
  "anonymous_id": "550e8400-e29b-41d4-a716-446655440000",
  "properties": {
    "platform": "ios",
    "app_version": "1.0.0"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `event_type` | string | Yes | One of the allowed event types below |
| `anonymous_id` | string (UUID) | Recommended | Device-level ID to stitch pre/post signup events |
| `properties` | object | No | Any extra key/value pairs relevant to the event |

### Allowed Event Types

| Event | When to fire |
|---|---|
| `app_install` | First ever app launch only (gate with SharedPreferences flag) |
| `signup_started` | User taps "Sign Up" button |
| `onboarding_skipped` | User skips onboarding |
| `onboarding_completed` | User completes onboarding flow |
| `session_started` | User starts a coaching session |
| `payment_started` | User opens the payment/subscription screen |
| `feature_viewed` | User views a specific feature (pass `"feature": "habits"` in properties) |

> Sending any other event type will be rejected with 400.

---

### Success Response

**HTTP 204 No Content**

```
HTTP/1.1 204 No Content
```

No body. Event was recorded. The app should never wait on this — fire and forget.

---

### Error Responses

**HTTP 400 — Invalid event type**
```json
{
  "detail": "Invalid event_type"
}
```
Cause: `event_type` is not in the allowed list. Check spelling. These errors are caught server-side and swallowed — the endpoint always returns 204 in production even if the event type is unknown. You will only see 400 during development.

---

**HTTP 422 — Validation error**
```json
{
  "detail": [
    {
      "type": "missing",
      "loc": ["body", "event_type"],
      "msg": "Field required"
    }
  ]
}
```
Cause: `event_type` field is missing from the body entirely.

---

**HTTP 429 — Rate limit exceeded**
```json
{
  "error": "Rate limit exceeded: 30 per 1 minute"
}
```
Cause: More than 30 events sent per minute from the same IP. This should never happen in normal usage. If it does, add a small delay between events.

---

### anonymous_id — Generate Once, Keep Forever

```dart
Future<String> getAnonymousId() async {
  final prefs = await SharedPreferences.getInstance();
  String? id = prefs.getString('anonymous_id');
  if (id == null) {
    id = const Uuid().v4();  // package: uuid
    await prefs.setString('anonymous_id', id);
  }
  return id;
}
```

This ID must:
- Be generated on first launch
- Survive app restarts and user logouts
- Be the same before and after signup (links funnel steps together)
- Only be cleared if the app is fully uninstalled

---

### Flutter Implementation

```dart
class AnalyticsService {
  static const _baseUrl = 'YOUR_BASE_URL';

  static Future<void> track(
    String eventType, {
    Map<String, dynamic> properties = const {},
  }) async {
    try {
      final anonymousId = await getAnonymousId();
      final token = await FirebaseAuth.instance.currentUser?.getIdToken();

      final headers = <String, String>{'Content-Type': 'application/json'};
      if (token != null) headers['Authorization'] = 'Bearer $token';

      await http.post(
        Uri.parse('$_baseUrl/api/v1/events'),
        headers: headers,
        body: jsonEncode({
          'event_type': eventType,
          'anonymous_id': anonymousId,
          'properties': properties,
        }),
      ).timeout(const Duration(seconds: 5));

    } catch (_) {
      // Never throw — analytics must never crash the app
    }
  }
}
```

**Where to fire each event:**

```dart
// main.dart — first launch only
final prefs = await SharedPreferences.getInstance();
if (!(prefs.getBool('app_install_fired') ?? false)) {
  AnalyticsService.track('app_install', properties: {
    'platform': Platform.isIOS ? 'ios' : 'android',
    'app_version': '1.0.0',
  });
  await prefs.setBool('app_install_fired', true);
}

// Sign up screen — when user taps Sign Up button
AnalyticsService.track('signup_started');

// After onboarding last step
AnalyticsService.track('onboarding_completed');

// When user opens payment/subscription screen
AnalyticsService.track('payment_started');

// When user views a specific feature
AnalyticsService.track('feature_viewed', properties: {'feature': 'habits'});
```

---

## 3. Voice Input (Optional)

**No backend changes needed.** Voice is handled entirely on-device. The transcribed text is sent to the existing message endpoints as normal text — the backend treats it identically.

### Recommended Package

```yaml
# pubspec.yaml
speech_to_text: ^6.6.0
```

### Required Permissions

**iOS** — add to `Info.plist`:
```xml
<key>NSMicrophoneUsageDescription</key>
<string>Kaya uses your microphone to let you speak your messages.</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>Kaya uses speech recognition to transcribe your voice messages.</string>
```

**Android** — add to `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
```

### Implementation

```dart
import 'package:speech_to_text/speech_to_text.dart';

class VoiceInputService {
  final SpeechToText _speech = SpeechToText();
  bool _isListening = false;

  Future<bool> initialize() async {
    return await _speech.initialize(
      onError: (error) => print('Speech error: $error'),
    );
  }

  Future<void> startListening({required Function(String text) onResult}) async {
    final available = await initialize();
    if (!available) {
      // Device does not support speech recognition
      return;
    }
    _isListening = true;
    await _speech.listen(
      onResult: (result) {
        if (result.finalResult) {
          _isListening = false;
          onResult(result.recognizedWords);
        }
      },
      listenFor: const Duration(seconds: 30),
      pauseFor: const Duration(seconds: 3),  // auto-stop after 3s silence
      localeId: 'en_US',
    );
  }

  void stopListening() {
    _speech.stop();
    _isListening = false;
  }

  bool get isListening => _isListening;
}
```

**Usage in chat screen:**

```dart
final _voiceService = VoiceInputService();

// Mic button onPressed
void onMicPressed() {
  if (_voiceService.isListening) {
    _voiceService.stopListening();
  } else {
    _voiceService.startListening(
      onResult: (text) {
        _messageController.text = text;
        // Optionally auto-send: sendMessage(text);
      },
    );
  }
}
```

**UX:** Add a microphone icon next to the text input. Tap to start recording, tap again to stop. Auto-stops after 3 seconds of silence.

---

## 4. Profile Picture Upload (Optional)

### Endpoint

```
POST /api/v1/me/profile-picture
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_id>
Content-Type: multipart/form-data
```

### Request

| Field | Type | Required | Description |
|---|---|---|---|
| `file` | File (form-data) | Yes | JPEG or PNG image, max 5 MB |

---

### Success Response

**HTTP 200 OK**

```json
{
  "profile_picture_url": "https://test.globalvin.co/kayatest/static/avatars/2610867f-ec66-4314-93fa-11537792a978.jpg"
}
```

| Field | Type | Description |
|---|---|---|
| `profile_picture_url` | string | Publicly accessible URL of the uploaded image. Store this and display it in the profile screen. |

The backend automatically resizes to max 512×512 px and compresses to JPEG at 85% quality before saving. Every upload overwrites the previous one — same URL each time.

---

### Error Responses

**HTTP 400 — Wrong file type**
```json
{
  "detail": "Only JPEG and PNG images are accepted"
}
```
Cause: File is not JPEG or PNG (e.g. GIF, WEBP, HEIC). Show: "Please select a JPEG or PNG image."

---

**HTTP 400 — File too large**
```json
{
  "detail": "File size must be 5 MB or less"
}
```
Cause: File exceeds 5 MB. Use `image_picker` with `imageQuality: 85` to compress before uploading. Show: "Image is too large. Please choose a smaller image."

---

**HTTP 401 — Not authenticated**
```json
{
  "detail": "Missing Bearer token"
}
```
or
```json
{
  "detail": "Invalid or expired token"
}
```
Cause: Token expired or missing `X-Device-Id` header. Refresh token and retry.

---

**HTTP 500 — Server error**
```json
{
  "detail": "Internal Server Error"
}
```
Cause: Unexpected error saving the file. Show: "Upload failed. Please try again."

---

### Required Package

```yaml
# pubspec.yaml
image_picker: ^1.1.2
```

### Flutter Implementation

```dart
import 'package:image_picker/image_picker.dart';

Future<void> uploadProfilePicture(BuildContext context) async {
  // Step 1: Pick image
  final picker = ImagePicker();
  final XFile? image = await picker.pickImage(
    source: ImageSource.gallery,
    maxWidth: 1024,
    maxHeight: 1024,
    imageQuality: 85,  // pre-compress before upload
  );
  if (image == null) return;  // user cancelled

  // Step 2: Upload
  try {
    final token = await FirebaseAuth.instance.currentUser?.getIdToken(true);
    final request = http.MultipartRequest(
      'POST',
      Uri.parse('$baseUrl/api/v1/me/profile-picture'),
    );
    request.headers['Authorization'] = 'Bearer $token';
    request.headers['X-Device-Id'] = deviceId;
    request.files.add(await http.MultipartFile.fromPath('file', image.path));

    final streamed = await request.send();
    final response = await http.Response.fromStream(streamed);
    final body = jsonDecode(response.body);

    // Step 3: Handle response
    if (response.statusCode == 200) {
      final url = body['profile_picture_url'] as String;
      // Update local state / provider with new URL
      // e.g. userProvider.updateProfilePicture(url);

    } else if (response.statusCode == 400) {
      final message = body['detail'] ?? 'Invalid image';
      _showError(context, message);

    } else if (response.statusCode == 401) {
      _showError(context, 'Session expired. Please log in again.');

    } else {
      _showError(context, 'Upload failed. Please try again.');
    }

  } catch (e) {
    _showError(context, 'No internet connection. Please try again.');
  }
}
```

**Where to use:** Profile screen → tap avatar → show bottom sheet with "Choose from Gallery" / "Take Photo" options → upload.

**Camera option (optional):**
```dart
// Replace ImageSource.gallery with ImageSource.camera for camera capture
final XFile? image = await picker.pickImage(source: ImageSource.camera);
```

**iOS permission for camera** — add to `Info.plist`:
```xml
<key>NSCameraUsageDescription</key>
<string>Kaya uses your camera to let you take a profile photo.</string>
<key>NSPhotoLibraryUsageDescription</key>
<string>Kaya accesses your photo library to let you choose a profile picture.</string>
```

**Android permission** — add to `AndroidManifest.xml`:
```xml
<uses-permission android:name="android.permission.CAMERA"/>
```
