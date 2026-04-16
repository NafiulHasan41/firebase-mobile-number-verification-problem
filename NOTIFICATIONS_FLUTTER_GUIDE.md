# Notifications — Flutter Integration Guide

**Status:** Live on `https://test.globalvin.co/kayatest` (as of 2026-04-17)
**Base URL:** `{BASE_URL}/api/v1/notifications`
**Auth:** Firebase ID token + single-device header (same pattern as all other endpoints)

This guide covers everything the Flutter app needs to implement push notifications (FCM), the in-app notifications drawer (bell icon), and deep-link navigation from notification taps.

---

## 1. Authentication Headers

Every request needs both headers:

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <unique_device_identifier>
Content-Type: application/json
```

---

## 2. Overview — How It Works

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Backend     │────▶│  FCM (Google)│────▶│  User's Phone   │
│  sends notif │     │  free service│     │  push appears    │
└──────┬───────┘     └────────��─────┘     └─────────────────┘
       │
       ▼
┌──────────────┐
│ notifications │  ◀── Every notification is also saved here
│ table (DB)    │      for the in-app bell icon drawer
└──────────────┘
```

1. Backend triggers a notification (event hook or hourly Celery scan)
2. If the user has a registered FCM token → push notification sent to phone
3. Notification is always logged to DB → powers the bell icon drawer
4. Flutter reads the drawer via `GET /notifications`

**Important:** Even if FCM delivery fails (bad token, no internet), the notification is still in the DB. The bell drawer always works.

---

## 3. Firebase Setup (Required)

### 3.1 Add Dependencies

```yaml
# pubspec.yaml
dependencies:
  firebase_core: ^3.0.0
  firebase_messaging: ^15.0.0
```

### 3.2 iOS Setup

1. **APNs Key** — Go to [Apple Developer Console](https://developer.apple.com/account/resources/authkeys/list) → Keys → Create key → Enable "Apple Push Notifications service (APNs)" → Download `.p8` file
2. **Upload to Firebase** — Firebase Console → Project Settings → Cloud Messaging → Upload APNs Authentication Key (`.p8` file + Key ID + Team ID)
3. **Xcode** — Open `ios/Runner.xcworkspace`:
   - Target → Signing & Capabilities → `+ Capability` → **Push Notifications**
   - Target → Signing & Capabilities → `+ Capability` → **Background Modes** → check "Remote notifications"
4. **Provisioning Profile** — Regenerate after enabling Push Notifications

### 3.3 Android Setup

- Android 13+ requires runtime permission. Add to `AndroidManifest.xml`:
  ```xml
  <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
  ```
- Request permission at runtime (see section 4.2)

### 3.4 Common Pitfalls

| Issue | Cause | Fix |
|---|---|---|
| iOS notifications not arriving | APNs key not uploaded to Firebase | Upload .p8 in Firebase Console → Cloud Messaging |
| iOS notifications not arriving | Push Notifications capability missing | Add in Xcode → Signing & Capabilities |
| Token is `null` on iOS | Permission not requested before `getToken()` | Call `requestPermission()` first |
| Works in debug, not release | Provisioning profile outdated | Regenerate after enabling Push Notifications |
| Android 13 not showing | Missing `POST_NOTIFICATIONS` permission | Add to manifest + request at runtime |

---

## 4. FCM Token Registration

### 4.1 Request Permission (iOS + Android 13+)

```dart
import 'package:firebase_messaging/firebase_messaging.dart';

Future<bool> requestNotificationPermission() async {
  final settings = await FirebaseMessaging.instance.requestPermission(
    alert: true,
    badge: true,
    sound: true,
  );
  return settings.authorizationStatus == AuthorizationStatus.authorized;
}
```

**When to call:** After login or after onboarding. Show a pre-permission screen explaining why notifications are useful, then trigger the system prompt.

### 4.2 Register Token with Backend

```
POST /api/v1/notifications/register-device-token
```

**Request:**
```json
{
  "token": "dGVzdC10b2tlbi0xMjM...",
  "platform": "ios"
}
```

**Response 200:**
```json
{ "status": "ok" }
```

**Errors:**
- `400` — `"platform must be 'ios' or 'android'"`
- `403` — Device conflict (use force-login)

### 4.3 Full Registration Flow

```dart
Future<void> registerDeviceToken() async {
  // 1. Request permission (iOS + Android 13+)
  final granted = await requestNotificationPermission();
  if (!granted) return; // User denied — app still works, just no push

  // 2. Get FCM token
  final token = await FirebaseMessaging.instance.getToken();
  if (token == null) return;

  // 3. Send to backend
  await api.post('/api/v1/notifications/register-device-token', body: {
    'token': token,
    'platform': Platform.isIOS ? 'ios' : 'android',
  });

  // 4. Listen for token refresh (Firebase rotates tokens periodically)
  FirebaseMessaging.instance.onTokenRefresh.listen((newToken) {
    api.post('/api/v1/notifications/register-device-token', body: {
      'token': newToken,
      'platform': Platform.isIOS ? 'ios' : 'android',
    });
  });
}
```

**Call `registerDeviceToken()` after every successful login.**

---

## 5. Handling Push Notifications

### 5.1 Foreground — App is Open

```dart
FirebaseMessaging.onMessage.listen((RemoteMessage message) {
  // Show in-app banner (don't rely on system notification)
  showInAppBanner(
    title: message.notification?.title ?? '',
    body: message.notification?.body ?? '',
  );

  // Refresh bell badge count
  refreshUnreadCount();
});
```

### 5.2 Background / Tap — User Taps Notification

```dart
// App was in background — user tapped notification
FirebaseMessaging.onMessageOpenedApp.listen((RemoteMessage message) {
  _handleNotificationTap(message.data);
});

// App was terminated — user tapped notification to open app
final initialMessage = await FirebaseMessaging.instance.getInitialMessage();
if (initialMessage != null) {
  _handleNotificationTap(initialMessage.data);
}
```

### 5.3 Deep Link Navigation

Every notification includes a `data` payload with an `action` field:

```dart
void _handleNotificationTap(Map<String, dynamic> data) {
  final action = data['action'];

  switch (action) {
    case 'open_home':
      navigate('/home');
      break;
    case 'open_session':
      navigate('/session/${data['session_id']}');
      break;
    case 'open_chat':
      navigate('/chat');
      break;
    case 'open_habits':
      navigate('/progress/habits');
      break;
    case 'open_mood':
      navigate('/mood');
      break;
    case 'open_progress':
      navigate('/progress');
      break;
    case 'open_balance':
      navigate('/payments/balance');
      break;
    case 'start_foundation':
      startFoundationSession();
      break;
    default:
      navigate('/home');
  }
}
```

### 5.4 Action Reference Table

| Action | Used by | Navigate to |
|---|---|---|
| `open_home` | Welcome | Home screen |
| `open_session` | Session reminder | Active session (has `session_id` in data) |
| `open_chat` | Check-in, re-engagement | Chat screen |
| `open_habits` | Streak protection | Habits list |
| `open_mood` | Mood insight (high/low) | Mood screen |
| `open_progress` | Milestone | Progress tab |
| `open_balance` | Payment success, renewal | Payment/balance screen |
| `start_foundation` | Feature discovery | Start foundation session |

---

## 6. Notifications Drawer (Bell Icon)

### 6.1 List Notifications

```
GET /api/v1/notifications?limit=50&offset=0
```

**Response 200:**
```json
{
  "notifications": [
    {
      "id": "6c950403-ef08-4d17-a5bc-1086b39d0ab7",
      "type": "mood_insight_high",
      "title": "Kaya Coach",
      "body": "Your energy has been higher this week. Keep that spirit!",
      "data": { "action": "open_mood" },
      "is_read": false,
      "created_at": "2026-04-16T17:59:06.852882+00:00"
    },
    {
      "id": "uuid-2",
      "type": "payment_success",
      "title": "Kaya",
      "body": "Payment successful. 3 sessions credited.",
      "data": { "action": "open_balance" },
      "is_read": true,
      "created_at": "2026-04-15T10:30:00+00:00"
    }
  ],
  "count": 2,
  "unread_count": 1
}
```

- Ordered **newest first**
- Paginated: `limit` (1–100, default 50) + `offset` (default 0)
- `unread_count` is the total across all pages (for the badge)

### 6.2 Unread Count (Bell Badge)

```
GET /api/v1/notifications/unread-count
```

**Response 200:**
```json
{ "count": 3 }
```

Lightweight — use this to update the badge without loading all notifications.

### 6.3 Mark as Read

```
POST /api/v1/notifications/{id}/read
```

**Response 200:**
```json
{ "status": "ok" }
```

**Errors:**
- `404` — Notification not found or doesn't belong to this user

### 6.4 Mark All as Read

```
POST /api/v1/notifications/read-all
```

**Response 200:**
```json
{ "status": "ok", "marked": 5 }
```

---

## 7. Dart Models

```dart
enum NotificationType {
  welcome,
  paymentSuccess,
  subscriptionRenewal,
  sessionReminder,
  streakProtection,
  moodInsightLow,
  moodInsightHigh,
  softReengagement,
  contextualReengagement,
  featureDiscovery,
  checkin,
  milestone,
}

class KayaNotification {
  final String id;
  final String type;
  final String title;
  final String body;
  final Map<String, dynamic>? data;
  final bool isRead;
  final DateTime createdAt;

  KayaNotification({
    required this.id,
    required this.type,
    required this.title,
    required this.body,
    this.data,
    required this.isRead,
    required this.createdAt,
  });

  factory KayaNotification.fromJson(Map<String, dynamic> json) {
    return KayaNotification(
      id: json['id'],
      type: json['type'],
      title: json['title'],
      body: json['body'],
      data: json['data'],
      isRead: json['is_read'],
      createdAt: DateTime.parse(json['created_at']),
    );
  }

  String? get action => data?['action'];
  String? get sessionId => data?['session_id'];
}

class NotificationListResponse {
  final List<KayaNotification> notifications;
  final int count;
  final int unreadCount;

  NotificationListResponse({
    required this.notifications,
    required this.count,
    required this.unreadCount,
  });

  factory NotificationListResponse.fromJson(Map<String, dynamic> json) {
    return NotificationListResponse(
      notifications: (json['notifications'] as List)
          .map((n) => KayaNotification.fromJson(n))
          .toList(),
      count: json['count'],
      unreadCount: json['unread_count'],
    );
  }
}
```

---

## 8. Service Class (Drop-in)

```dart
class NotificationService {
  final ApiClient api;

  NotificationService(this.api);

  /// Register FCM token — call after every login
  Future<void> registerToken(String token, String platform) async {
    await api.post('/api/v1/notifications/register-device-token', body: {
      'token': token,
      'platform': platform,
    });
  }

  /// Fetch notification list (paginated, newest first)
  Future<NotificationListResponse> list({int limit = 50, int offset = 0}) async {
    final r = await api.get('/api/v1/notifications?limit=$limit&offset=$offset');
    return NotificationListResponse.fromJson(r.data);
  }

  /// Get unread count for bell badge
  Future<int> unreadCount() async {
    final r = await api.get('/api/v1/notifications/unread-count');
    return r.data['count'];
  }

  /// Mark single notification as read
  Future<void> markRead(String notificationId) async {
    await api.post('/api/v1/notifications/$notificationId/read');
  }

  /// Mark all notifications as read
  Future<int> markAllRead() async {
    final r = await api.post('/api/v1/notifications/read-all');
    return r.data['marked'];
  }
}
```

---

## 9. UI Patterns

### 9.1 Bell Icon with Badge

```dart
class NotificationBell extends StatelessWidget {
  final NotificationService notifService;
  final VoidCallback onTap;

  const NotificationBell({required this.notifService, required this.onTap});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<int>(
      future: notifService.unreadCount(),
      builder: (_, snap) {
        final count = snap.data ?? 0;
        return Badge(
          isLabelVisible: count > 0,
          label: Text('$count'),
          child: IconButton(
            icon: const Icon(Icons.notifications_outlined),
            onPressed: onTap,
          ),
        );
      },
    );
  }
}
```

### 9.2 Notifications Drawer

```dart
class NotificationsDrawer extends StatefulWidget {
  final NotificationService notifService;

  const NotificationsDrawer({required this.notifService});

  @override
  State<NotificationsDrawer> createState() => _NotificationsDrawerState();
}

class _NotificationsDrawerState extends State<NotificationsDrawer> {
  List<KayaNotification> items = [];
  bool loading = true;

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    final resp = await widget.notifService.list();
    setState(() {
      items = resp.notifications;
      loading = false;
    });
  }

  IconData _iconForType(String type) {
    switch (type) {
      case 'welcome':              return Icons.celebration;
      case 'payment_success':      return Icons.payment;
      case 'subscription_renewal': return Icons.autorenew;
      case 'session_reminder':     return Icons.timer;
      case 'streak_protection':    return Icons.local_fire_department;
      case 'mood_insight_high':    return Icons.trending_up;
      case 'mood_insight_low':     return Icons.trending_down;
      case 'soft_reengagement':    return Icons.chat_bubble_outline;
      case 'contextual_reengagement': return Icons.chat_bubble_outline;
      case 'feature_discovery':    return Icons.star_outline;
      case 'checkin':              return Icons.favorite;
      default:
        if (type.startsWith('milestone_')) return Icons.emoji_events;
        return Icons.notifications;
    }
  }

  String _timeAgo(DateTime dt) {
    final diff = DateTime.now().difference(dt);
    if (diff.inMinutes < 1) return 'just now';
    if (diff.inMinutes < 60) return '${diff.inMinutes}m ago';
    if (diff.inHours < 24) return '${diff.inHours}h ago';
    if (diff.inDays < 7) return '${diff.inDays}d ago';
    return '${dt.month}/${dt.day}';
  }

  @override
  Widget build(BuildContext context) {
    if (loading) return const Center(child: CircularProgressIndicator());

    if (items.isEmpty) {
      return const Center(child: Text('No notifications yet'));
    }

    return Column(
      children: [
        // "Mark all as read" button
        if (items.any((n) => !n.isRead))
          TextButton(
            onPressed: () async {
              await widget.notifService.markAllRead();
              _load(); // Refresh
            },
            child: const Text('Mark all as read'),
          ),

        Expanded(
          child: ListView.builder(
            itemCount: items.length,
            itemBuilder: (_, i) {
              final n = items[i];
              return ListTile(
                leading: Icon(_iconForType(n.type)),
                title: Text(n.title),
                subtitle: Text(n.body, maxLines: 2, overflow: TextOverflow.ellipsis),
                trailing: Text(_timeAgo(n.createdAt),
                    style: Theme.of(context).textTheme.bodySmall),
                tileColor: n.isRead ? null : Colors.blue.withOpacity(0.05),
                onTap: () async {
                  if (!n.isRead) {
                    await widget.notifService.markRead(n.id);
                  }
                  _handleNotificationTap(n.data ?? {});
                },
              );
            },
          ),
        ),
      ],
    );
  }
}
```

---

## 10. Notification Types Reference

The backend sends these notification types. Each has a dedup window — the same type won't be sent twice within that window.

| Type | Title | Example Body | Trigger | Dedup |
|---|---|---|---|---|
| `welcome` | Kaya | Welcome to Kaya! | Signup | Once ever |
| `payment_success` | Kaya | Payment successful. 3 sessions credited. | Purchase/subscribe | None |
| `subscription_renewal` | Kaya | Your monthly plan renewed. 3 new sessions available. | Monthly renewal | None |
| `session_reminder` | Kaya Coach | Your session has 4 hours remaining. Let's talk with Kaya. | Active session ≤4hrs left | 24h |
| `streak_protection` | Kaya Coach | Your 7-day streak is at risk. A quick check-in keeps it going. | Evening, no habit log today | 24h |
| `mood_insight_high` | Kaya Coach | Your energy has been higher this week. Keep that spirit! | Weekly avg ≥2 points up | 7 days |
| `mood_insight_low` | Kaya Coach | Your energy has been lower this week. A short rest might help. | Weekly avg ≥2 points down | 7 days |
| `soft_reengagement` | Kaya Coach | It's been a while. Let's chat with Kaya! | 2+ days inactive | 3 days |
| `contextual_reengagement` | Kaya Coach | Busy week? Want to chat with Kaya? | 5+ days inactive | 7 days |
| `feature_discovery` | Kaya Coach | Your free foundation session is waiting. Let's chat with Kaya. | Intro done, no foundation started | 3 days |
| `checkin` | Kaya Coach | Hey {name}, how are you doing with your goal to {goal}? | Scheduled check-in due | Per schedule |
| `milestone_{N}` | Kaya Coach | You've completed {N} sessions. Keep going! | 5/10/25/50/100 sessions | Once per milestone |

---

## 11. When to Refresh

| Event | Refresh |
|---|---|
| App opens / resumes from background | `unreadCount()` for badge |
| User opens notification drawer | `list()` |
| User taps a notification | `markRead(id)` then navigate |
| User taps "Mark all as read" | `markAllRead()` then `list()` |
| Foreground push received (`onMessage`) | `unreadCount()` for badge |
| After login | `registerToken()` |

---

## 12. Background Message Handler (Required for iOS)

```dart
// This must be a top-level function (not inside a class)
@pragma('vm:entry-point')
Future<void> _firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  // iOS needs this to display notifications when app is in background
  // No navigation here — that's handled by onMessageOpenedApp
}

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();

  // Register background handler
  FirebaseMessaging.onBackgroundMessage(_firebaseMessagingBackgroundHandler);

  runApp(MyApp());
}
```

---

## 13. Full Initialization Example

```dart
class AppInitializer {
  static Future<void> initNotifications(ApiClient api) async {
    // 1. Request permission
    final settings = await FirebaseMessaging.instance.requestPermission(
      alert: true,
      badge: true,
      sound: true,
    );

    if (settings.authorizationStatus != AuthorizationStatus.authorized) {
      return; // User denied — skip registration
    }

    // 2. Get and register token
    final token = await FirebaseMessaging.instance.getToken();
    if (token != null) {
      await api.post('/api/v1/notifications/register-device-token', body: {
        'token': token,
        'platform': Platform.isIOS ? 'ios' : 'android',
      });
    }

    // 3. Listen for token refresh
    FirebaseMessaging.instance.onTokenRefresh.listen((newToken) {
      api.post('/api/v1/notifications/register-device-token', body: {
        'token': newToken,
        'platform': Platform.isIOS ? 'ios' : 'android',
      });
    });

    // 4. Foreground notification handler
    FirebaseMessaging.onMessage.listen((message) {
      // Show in-app banner
      showInAppNotification(
        title: message.notification?.title ?? '',
        body: message.notification?.body ?? '',
      );
      // Refresh badge
      NotificationService(api).unreadCount().then((count) {
        updateBadge(count);
      });
    });

    // 5. Background tap handler
    FirebaseMessaging.onMessageOpenedApp.listen((message) {
      _handleNotificationTap(message.data);
    });

    // 6. App was terminated — opened from notification
    final initial = await FirebaseMessaging.instance.getInitialMessage();
    if (initial != null) {
      _handleNotificationTap(initial.data);
    }
  }
}
```

Call `AppInitializer.initNotifications(api)` after successful login.

---

## 14. Error Handling

| HTTP Code | Meaning | Action |
|---|---|---|
| `200` | Success | Process response |
| `400` | Bad request (invalid platform) | Show error |
| `401` | Token expired | Refresh Firebase token, retry |
| `403` | Device conflict | Show force-login prompt |
| `404` | Notification not found | Ignore (may have been deleted) |
| `422` | Missing headers | Check Authorization + X-Device-Id |
| `500` | Server error | Retry with backoff |

---

