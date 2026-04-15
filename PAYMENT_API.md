# Kaya — Payment API Guide
_For the Flutter developer. Everything needed to implement the paywall, purchases, and subscription flow._
_Base URL: `http://165.232.55.80:8000`_

---

## Overview

Kaya uses **PayPal** for payments. There are two payment types:

| Type | Description |
|---|---|
| One-time purchase | Buy a single session or a bundle of 3 sessions |
| Monthly subscription | $26/month — 3 sessions per billing period |

### Free Sessions (No Payment Required)

Every user gets these for free:
- **Intro session** — always free, no credits needed
- **First foundation session** — the first attempt is free regardless of outcome

Once the free foundation slot is used (even if force-ended), the user must pay to continue.

---

## 🚨 IMPORTANT — Which endpoint for which plan

**This is the #1 source of bugs. Read it carefully.**

| Plan | `sessions` in pricing | Endpoint to call | Allowed while monthly sub is active? |
|---|---|---|---|
| `single_session` | `1` | **`POST /create-order`** | ✅ **YES — always** |
| `bundle_3` | `3` | **`POST /create-order`** | ✅ **YES — always** |
| `subscription_monthly` | `0` | **`POST /create-subscription`** | ❌ NO — returns `409` if user already has an active sub |

### Rule in plain English

> A user with an **active monthly subscription** can **still buy** a single session or a 3-session bundle at any time. Those purchases land in `paid_balance` and stack on top of the subscription's `sessions_remaining`. The user's `total_usable` = subscription quota + bundle credits.
>
> The **only** thing blocked while a subscription is active is creating **another** subscription. That's what the `409` on `/create-subscription` means — it is NOT a "no more purchases allowed" error.

### Routing rule for Flutter

Read `sessions` from the pricing response and branch on it:

```dart
if (plan.sessions > 0) {
  // single_session, bundle_3  → one-time purchase
  await api.post('/api/v1/payments/create-order', body: {'plan_key': plan.key});
} else {
  // subscription_monthly  → recurring subscription
  await api.post('/api/v1/payments/create-subscription', body: {'plan_key': plan.key});
}
```

**Never** send `bundle_3` or `single_session` to `/create-subscription`. If you do, the user will see "You already have an active subscription" when they try to buy a bundle — which looks like "bundle buying is broken" but is really a wrong-endpoint bug on the frontend.

### What the backend actually checks

- `POST /create-order` — only rejects if the `plan_key` is a subscription plan (`sessions == 0`). **Does NOT check for existing subscriptions.** Safe to call regardless of sub status.
- `POST /create-subscription` — rejects with `409` if the user already has an `active` or `cancelled` (within period) subscription.

---

## Sandbox Test Credentials (for development only)

When the user taps "Subscribe" or "Buy" during development, the app opens a PayPal **sandbox** page. You need a sandbox **buyer** account to approve the payment — real PayPal credentials will NOT work against the sandbox environment.

Use this test buyer account on the PayPal approval page:

| Field | Value |
|---|---|
| Email | `sb-i47xnv29958827@personal.example.com` |
| Password | `OM<cB3q]` |

**How it works during testing:**

1. User taps "Subscribe" / "Buy" in the Kaya app
2. The app opens the PayPal approval URL in the `PayPalWebView`
3. On PayPal's page, sign in with the sandbox buyer credentials above
4. Click **Agree & Subscribe** (subscriptions) or **Continue** → **Pay Now** (one-time)
5. PayPal redirects to `kaya://payment/success?token=...` — the `PayPalWebView` intercepts and closes
6. Flutter calls `/capture-order` (one-time) or waits for the webhook (subscription)

**Important notes:**

- This is a **sandbox** account — no real money is charged
- The sandbox buyer always has enough test balance to complete any transaction
- Do NOT ship these credentials in the production app build
- If the PayPal page says "Your account is locked" or "Cannot log in", the sandbox buyer was rate-limited by too many rapid attempts — wait 10 minutes and retry
- If the PayPal page shows in Arabic or another language, the backend already forces `locale.x=en_US` on the approval URL — if you see another locale, report it so we can investigate

**For production builds:**

Real users will sign in with their own real PayPal credentials (or use the card-entry path via "Create an Account"). Production uses `api.paypal.com`; sandbox uses `api-m.sandbox.paypal.com`. The backend switches environments via an env var — the Flutter app doesn't need to know which environment is active.

---

## The Paywall Decision — Use This Endpoint

**This is the single source of truth for whether to show the paywall.**

```
GET /api/v1/payments/balance
Headers: Authorization, X-Device-Id
```

### Response

```json
{
  "paid_balance": 0,
  "subscription": null,
  "free_sessions": {
    "intro_completed": true,
    "foundation_free_used": true
  },
  "total_usable": 0,
  "sessions_taken": 2,
  "paywall_required": true
}
```

### Field Reference

| Field | Type | Description |
|---|---|---|
| `paid_balance` | `int` | Number of individually purchased session credits remaining — **only counts one-off single-session and bundle buys**, NOT subscription sessions |
| `subscription` | `object \| null` | Active subscription details, or null if none |
| `subscription.status` | `string` | `"active"` \| `"cancelled"` \| `"past_due"` \| `"failed"` |
| `subscription.plan_key` | `string` | e.g. `"subscription_monthly"` |
| `subscription.sessions_remaining` | `int` | Sessions left in current billing period |
| `subscription.renews_at` | `string \| null` | ISO timestamp of next renewal |
| `free_sessions.intro_completed` | `bool` | Whether intro was properly completed with consent |
| `free_sessions.foundation_free_used` | `bool` | Whether the free foundation slot has been consumed |
| `total_usable` | `int` | **Total sessions the user can start right now** = `paid_balance` + `subscription.sessions_remaining`. **Always use this field for the credits display and the paywall check** |

> ### ⚠️ IMPORTANT — which field to display for "Credits"
>
> **Always display `total_usable`.** Never display `paid_balance` as "your credits" — it only counts one-off purchases, so a user with an **active monthly subscription** (and no one-off buys) will see `paid_balance: 0` while actually having 3 usable sessions. Showing `paid_balance` in that case produces a wrong "0 credits" display and confused users.
>
> Example response for a subscription-only user:
> ```json
> {
>   "paid_balance": 0,                       // ← DO NOT display this
>   "subscription": {
>     "status": "active",
>     "sessions_remaining": 3                // ← subscription pool
>   },
>   "total_usable": 3                        // ← DISPLAY THIS as "Credits"
> }
> ```
>
> This applies to **both** `GET /api/v1/payments/balance` **and** `GET /api/v1/me/home` (where the same `credits` shape is nested inside the response).
| `sessions_taken` | `int` | Total coaching sessions ever started (all types, all statuses) |
| `paywall_required` | `bool` | **Show paywall when this is true** |

### Paywall Logic

```dart
final balance = await api.getBalance();

if (balance['paywall_required'] == true) {
  // User has used their free sessions and has no credits
  showPaywall();
} else {
  // User can start a session
  allowSessionStart();
}
```

`paywall_required` is `true` when ALL of these are true:
1. Intro is completed (`intro_completed: true`)
2. Free foundation slot is used (`foundation_free_used: true`)
3. No credits available (`total_usable == 0`)

### When `foundation_free_used` Becomes True

The free foundation slot is considered "used" the moment ANY foundation session is started — **regardless of whether it completed, was force-ended, or expired**. This prevents users from bypassing payment by repeatedly force-ending.

---

## Pricing — Show Before Paywall

```
GET /api/v1/payments/pricing
No auth required
```

### Response

```json
{
  "pricing": [
    {
      "key": "single_session",
      "label": "Single Session",
      "amount": 16.0,
      "amount_cents": 1600,
      "currency": "USD",
      "sessions": 1,
      "perks": [
        "1 coaching session",
        "Personalized AI health coach",
        "Goal & habit tracking"
      ]
    },
    {
      "key": "bundle_3",
      "label": "3-Session Bundle",
      "amount": 35.0,
      "amount_cents": 3500,
      "currency": "USD",
      "sessions": 3,
      "perks": [
        "3 coaching sessions",
        "Save vs. single sessions",
        "Goal & habit tracking"
      ]
    },
    {
      "key": "subscription_monthly",
      "label": "Monthly Subscription (3 sessions)",
      "amount": 26.0,
      "amount_cents": 2600,
      "currency": "USD",
      "sessions": 0,
      "perks": [
        "3 sessions per month",
        "Auto-renews monthly",
        "Cancel anytime"
      ]
    }
  ]
}
```

**Note:** `sessions: 0` means it is a subscription plan, not a one-time purchase.

**Note:** `perks` is an ordered array of feature strings set by the admin. Render as a bullet list on the paywall/pricing screen. The array may be empty `[]` if no perks have been configured yet — handle gracefully.

---

## One-Time Purchase Flow

Use this for single session or bundle purchases.

### Step 1 — Create Order

```
POST /api/v1/payments/create-order
Headers: Authorization, X-Device-Id
Body: { "plan_key": "single_session" }
       OR
       { "plan_key": "bundle_3" }
```

**Response:**
```json
{
  "order_id": "9XW12345AB678901C",
  "approval_url": "https://www.sandbox.paypal.com/checkoutnow?token=..."
}
```

### Step 2 — Open PayPal Approval URL

Open `approval_url` in an in-app WebView. PayPal redirects to `kaya://payment/success?token=<order_id>&PayerID=<id>` on approval or `kaya://payment/cancelled` on cancel.

> ⚠️ **Important:** The `kaya://` scheme is a custom deep link — NOT a web URL. A plain WebView will fail with `net::ERR_UNKNOWN_URL_SCHEME` when PayPal redirects to it. You **must** intercept the navigation inside the WebView before it tries to load that URL.
>
> See the **[WebView Deep-Link Handling (Flutter)](#webview-deep-link-handling-flutter)** section below for the full implementation. That section is shared by both one-time purchases and subscriptions.

### Step 3 — Capture Payment

```
POST /api/v1/payments/capture-order
Headers: Authorization, X-Device-Id
Body: { "order_id": "9XW12345AB678901C" }
```

**Response:**
```json
{
  "status": "completed",
  "sessions_credited": 1,
  "new_balance": 1
}
```

After this, `paid_balance` on `/payments/balance` will increase by the sessions purchased. Safe to call multiple times — idempotent.

---

## Subscription Flow

Use this for the monthly plan (`subscription_monthly`).

### How session quotas work

- On **initial subscription**, the user receives the plan's sessions (3 for the monthly plan).
- On each **successful renewal**, `sessions_remaining` is **reset** to the plan's monthly quota — **unused sessions from the previous month do NOT carry over**. It is one month, one fresh quota. Use it or lose it.
- On **cancellation**, the user keeps their remaining sessions until `current_period_end`. After the period ends, those sessions become unusable (zeroed out by a background sweep within ~5 minutes).
- A user **cannot start a new subscription** while a previous cancelled one still has usable sessions inside its paid period. Either wait until the period ends, or consume the remaining sessions. The re-subscribe error response carries `available_until` so Flutter can show a clear message.

### Step 1 — Create Subscription

```
POST /api/v1/payments/create-subscription
Headers: Authorization, X-Device-Id
Body: { "plan_key": "subscription_monthly" }
```

**Response:**
```json
{
  "subscription_id": "I-BW452GLLEP1G",
  "approval_url": "https://www.sandbox.paypal.com/webapps/billing/subscriptions/..."
}
```

### Step 2 — Open PayPal Approval URL

Same deep-link handling as one-time purchase — see **[WebView Deep-Link Handling (Flutter)](#webview-deep-link-handling-flutter)** below. The only difference: **there is no capture step for subscriptions**. PayPal fires a webhook to the backend which automatically credits the sessions — after the WebView returns, just refresh `/balance`.

**Note:** There may be a 2–5 second delay between PayPal approval and the backend receiving the webhook. Refresh balance after a short wait if `sessions_remaining` still shows 0.

### Step 3 — Cancel Subscription

```
POST /api/v1/payments/cancel-subscription
Headers: Authorization, X-Device-Id
No body required
```

**Response:**
```json
{
  "status": "cancelled",
  "message": "Subscription cancelled. Sessions remain until end of billing period."
}
```

The user keeps remaining sessions until the billing period ends. The `subscription.status` changes to `"cancelled"`.

---

## WebView Deep-Link Handling (Flutter)

This section is shared by both the one-time purchase flow and the subscription flow. Read it once and apply it to both.

### The error you'll hit without this

If you open `approval_url` in a standard WebView and just wait for navigation to finish, you'll see this error after the user completes payment on PayPal:

```
Web page not available

The web page at:
kaya://payment/success?token=8L625562GL378833K&PayerID=32RQS3Z5YXTG2
could not be loaded because:

net::ERR_UNKNOWN_URL_SCHEME
```

### Why it happens

After the user approves the payment on PayPal's page, PayPal sends the WebView an HTTP 302 redirect with a `Location` header pointing to `kaya://payment/success?token=...&PayerID=...`.

A WebView only knows how to load `http://` and `https://` URLs. `kaya://` is a **custom deep-link scheme** — it is not a web URL, and the WebView has no idea how to "load" it. So it fails with `ERR_UNKNOWN_URL_SCHEME` and the user is stuck on a broken error page.

The fix is to **intercept the navigation** inside the WebView and handle the `kaya://` URL yourself — close the WebView, parse the query parameters, and call the backend — instead of letting the WebView try to load it.

### The flow

```
User taps "Subscribe" / "Buy"
    │
    ▼
Flutter calls POST /create-order  (or /create-subscription)
    │
    ▼
Backend returns approval_url
    │
    ▼
Flutter opens approval_url in InAppWebView
    │
    ▼
User approves on PayPal's page
    │
    ▼
PayPal redirects to kaya://payment/success?token=...
    │
    ▼
shouldOverrideUrlLoading callback fires  ← INTERCEPT HERE
    │
    ├─ URL starts with "kaya://"?  → YES
    │     │
    │     ▼
    │  Return NavigationActionPolicy.CANCEL
    │  Close the WebView
    │  Call POST /capture-order  (one-time only)
    │  or refresh /balance        (subscriptions)
    │
    └─ Otherwise → NavigationActionPolicy.ALLOW (normal PayPal navigation)
```

### Recommended package

Use **`flutter_inappwebview`** — it has the most reliable `shouldOverrideUrlLoading` callback and handles PayPal's cookies correctly across redirects (including 3D-Secure chains). Install:

```yaml
dependencies:
  flutter_inappwebview: ^6.1.5
```

### Drop-in widget

Copy this into `lib/widgets/paypal_webview.dart`. It's reusable for both purchase flows.

```dart
import 'package:flutter/material.dart';
import 'package:flutter_inappwebview/flutter_inappwebview.dart';

/// Opens a PayPal approval URL in a WebView, intercepts the kaya:// deep link,
/// and returns a PayPalResult to the caller.
class PayPalWebView extends StatefulWidget {
  final String approvalUrl;
  const PayPalWebView({super.key, required this.approvalUrl});

  @override
  State<PayPalWebView> createState() => _PayPalWebViewState();
}

class _PayPalWebViewState extends State<PayPalWebView> {
  bool _loading = true;
  bool _handled = false; // prevent double-pop if PayPal redirects twice

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Complete Payment'),
        leading: IconButton(
          icon: const Icon(Icons.close),
          onPressed: () =>
              Navigator.of(context).pop(const PayPalResult.cancelled()),
        ),
      ),
      body: Stack(
        children: [
          InAppWebView(
            initialUrlRequest: URLRequest(url: WebUri(widget.approvalUrl)),
            initialSettings: InAppWebViewSettings(
              javaScriptEnabled: true,
              useShouldOverrideUrlLoading: true, // ← critical
              thirdPartyCookiesEnabled: true,
            ),

            // THIS is the fix for net::ERR_UNKNOWN_URL_SCHEME
            shouldOverrideUrlLoading: (controller, navigationAction) async {
              final url = navigationAction.request.url.toString();

              // Intercept our custom deep-link BEFORE the WebView tries to load it
              if (url.startsWith('kaya://')) {
                if (_handled) return NavigationActionPolicy.CANCEL;
                _handled = true;

                final uri = Uri.parse(url);
                late PayPalResult result;

                if (uri.host == 'payment' && uri.path == '/success') {
                  // PayPal echoes the order_id back as `token` on success.
                  // You already have the order_id from create-order, so you
                  // don't strictly need it — it's just here for confirmation.
                  final token = uri.queryParameters['token'];
                  final payerId = uri.queryParameters['PayerID'];
                  result = PayPalResult.success(
                    token: token,
                    payerId: payerId,
                  );
                } else if (uri.host == 'payment' && uri.path == '/cancelled') {
                  result = const PayPalResult.cancelled();
                } else {
                  result = PayPalResult.error('Unknown deep link: $url');
                }

                if (mounted) Navigator.of(context).pop(result);
                return NavigationActionPolicy.CANCEL; // ← don't let WebView try to load kaya://
              }

              // Everything else (PayPal's own https:// redirects) passes through
              return NavigationActionPolicy.ALLOW;
            },

            onLoadStop: (controller, url) {
              if (mounted) setState(() => _loading = false);
            },
            onReceivedError: (controller, request, error) {
              if (_handled) return;
              _handled = true;
              if (mounted) {
                Navigator.of(context).pop(
                  PayPalResult.error('Network error: ${error.description}'),
                );
              }
            },
          ),
          if (_loading) const Center(child: CircularProgressIndicator()),
        ],
      ),
    );
  }
}

/// Sealed result type so the caller is forced to handle all three outcomes.
sealed class PayPalResult {
  const PayPalResult();
  const factory PayPalResult.success({String? token, String? payerId}) =
      PayPalSuccess;
  const factory PayPalResult.cancelled() = PayPalCancelled;
  const factory PayPalResult.error(String message) = PayPalError;
}

class PayPalSuccess extends PayPalResult {
  final String? token;
  final String? payerId;
  const PayPalSuccess({this.token, this.payerId});
}

class PayPalCancelled extends PayPalResult {
  const PayPalCancelled();
}

class PayPalError extends PayPalResult {
  final String message;
  const PayPalError(this.message);
}
```

### Using it — one-time purchase

```dart
Future<void> buyPlan(String planKey) async {
  // 1. Create the order on the backend
  final order = await api.post(
    '/api/v1/payments/create-order',
    body: {'plan_key': planKey},
  );
  final orderId = order['order_id'] as String;
  final approvalUrl = order['approval_url'] as String;

  // 2. Open PayPal approval in the WebView and wait for the user
  final result = await Navigator.of(context).push<PayPalResult>(
    MaterialPageRoute(
      builder: (_) => PayPalWebView(approvalUrl: approvalUrl),
    ),
  );

  // 3. Handle the outcome
  switch (result) {
    case PayPalSuccess():
      // Call capture-order to finalize the charge.
      // Use the orderId you already have — no need to read `token` from the URL.
      final capture = await api.post(
        '/api/v1/payments/capture-order',
        body: {'order_id': orderId},
      );
      showSnackbar('Purchased! ${capture['sessions_credited']} sessions added.');
      await refreshBalance();

    case PayPalCancelled():
      showSnackbar('Payment cancelled.');

    case PayPalError(:final message):
      showError('Payment failed: $message');

    case null:
      // User dismissed the WebView without a result
      showSnackbar('Payment cancelled.');
  }
}
```

### Using it — subscription

Subscriptions are almost identical, except **there is no capture step** — the backend receives a PayPal webhook and credits sessions automatically.

```dart
Future<void> subscribe() async {
  // 1. Create the subscription on the backend
  final sub = await api.post(
    '/api/v1/payments/create-subscription',
    body: {'plan_key': 'subscription_monthly'},
  );
  final approvalUrl = sub['approval_url'] as String;

  // 2. Open PayPal approval in the WebView
  final result = await Navigator.of(context).push<PayPalResult>(
    MaterialPageRoute(
      builder: (_) => PayPalWebView(approvalUrl: approvalUrl),
    ),
  );

  // 3. Handle the outcome
  switch (result) {
    case PayPalSuccess():
      // NO capture-order call — the webhook credits the sessions.
      // Give the backend 2-5 seconds to receive the webhook, then refresh.
      await Future.delayed(const Duration(seconds: 3));
      await refreshBalance();
      showSnackbar('Subscription active!');

    case PayPalCancelled():
      showSnackbar('Subscription cancelled.');

    case PayPalError(:final message):
      showError('Subscription failed: $message');

    case null:
      showSnackbar('Subscription cancelled.');
  }
}
```

### Key points to remember

| Point | Why it matters |
|---|---|
| `useShouldOverrideUrlLoading: true` must be set in `InAppWebViewSettings` | Otherwise the callback is never called |
| Return `NavigationActionPolicy.CANCEL` for `kaya://` URLs | Stops the WebView from trying to load a non-http URL → no more `ERR_UNKNOWN_URL_SCHEME` |
| `_handled` flag guards against double-handling | PayPal occasionally redirects twice in quick succession; without the guard, you'll pop the route twice and crash |
| Non-`kaya://` URLs return `NavigationActionPolicy.ALLOW` | PayPal's internal https redirects (login, 3DS, etc.) must pass through normally |
| For one-time purchases you already have the `order_id` | The `token` query param in the success URL is just PayPal echoing the order_id back — you don't need to parse it |
| For subscriptions, refresh `/balance` after a short delay | The backend webhook may take 2–5 seconds to arrive |
| `webview_flutter` also works, but `flutter_inappwebview` is recommended | Better cookie handling for PayPal, more reliable across iOS + Android |

### What NOT to do

| Wrong | Why |
|---|---|
| Use a plain `WebView` without `shouldOverrideUrlLoading` | Causes `net::ERR_UNKNOWN_URL_SCHEME` on the `kaya://` redirect |
| Try to `Uri.parse(url)` **after** the WebView errors out | Too late — the error is the symptom, the navigation must be intercepted **before** it happens |
| Open `approval_url` in an external browser via `url_launcher` | Works, but forces the user out of the Kaya app and requires extra Android/iOS manifest setup to handle the `kaya://` return |
| Listen for `onLoadStart`/`onLoadStop` to detect the success URL | Unreliable — `onLoadStart` may fire with `kaya://` but you can't cancel the navigation from there. Use `shouldOverrideUrlLoading` |

---

## Subscription States

| `subscription.status` | Meaning | What to show | Cancel button? |
|---|---|---|---|
| `"active"` | Subscription running, sessions available | Sessions remaining + renewal date | ✅ Yes |
| `"cancelled"` | User cancelled, still has sessions until period ends | "Cancelled — sessions available until [renews_at]" | ❌ No |
| `"past_due"` | Renewal charge failed, PayPal retrying | Warning banner: "Payment failed — update your payment method". Sessions still usable until retries exhaust | ✅ Yes (and "Update payment method" link) |
| `"failed"` | **First** charge failed on a brand-new subscription — money was never taken | Error: "Initial payment failed — try again with a different card". Offer retry by creating a new subscription | ❌ No (and offer "Try again") |
| `null` (no subscription) | No active subscription | Show purchase options | — |

### Required Flutter UI

An Account / Settings screen must render the subscription state from `GET /balance`:

```
Monthly Subscription
Status: <status>
<N> sessions remaining
Renews: <renews_at>   (or "Available until" for cancelled)
[ Cancel Subscription ] (only when status == "active" or "past_due")
```

**Cancel confirmation dialog** (when user taps Cancel):

> Cancel Subscription?
>
> You'll keep your <N> remaining sessions until <renews_at>. After that, you'll need to subscribe again to continue.
>
> [ Keep ]   [ Cancel ]

Call `POST /api/v1/payments/cancel-subscription` only after confirmation. Refresh `GET /balance` afterward so the UI reflects the new `cancelled` status.

### Note on PayPal Approval Cancellation

If the user closes the PayPal approval WebView without completing approval, or their card is declined on PayPal's page, **no subscription is created** and no money is charged. The user can simply tap "Subscribe" again to retry — there's nothing to clean up on the backend.

---

## Payment History

```
GET /api/v1/payments/history
Headers: Authorization, X-Device-Id
```

**Response:**
```json
{
  "payments": [
    {
      "type": "single_session",
      "provider": "paypal",
      "amount": 16.0,
      "currency": "USD",
      "status": "completed",
      "sessions_credited": 1,
      "date": "2026-04-08T10:00:00+00:00"
    }
  ]
}
```

Returns the last 20 **completed** payments for the user. Use this for a billing history screen.

**Note:** Abandoned orders (orders that were created but never paid for — `status='pending'`) are filtered out by the backend. The user will only see payments that were actually charged. No client-side filtering needed.

---

## Complete Flow Diagram

```
App Launch
    │
    ▼
GET /api/v1/payments/balance
    │
    ├─ paywall_required: false ──► Allow session start
    │
    └─ paywall_required: true
           │
           ▼
    GET /api/v1/payments/pricing  (show plan options)
           │
           ├─ User picks one-time plan
           │       │
           │       ▼
           │  POST /api/v1/payments/create-order
           │       │
           │       ▼
           │  Open PayPal approval URL
           │       │
           │       ▼
           │  POST /api/v1/payments/capture-order
           │       │
           │       ▼
           │  GET /api/v1/payments/balance  (confirm new_balance > 0)
           │       │
           │       ▼
           │  Allow session start
           │
           └─ User picks subscription
                   │
                   ▼
             POST /api/v1/payments/create-subscription
                   │
                   ▼
             Open PayPal approval URL
                   │
                   ▼
             Wait for webhook → GET /api/v1/payments/balance
                   │
                   ▼
             Allow session start
```

---

## Quick Reference — Endpoint Summary

| Endpoint | Auth | Method | Purpose |
|---|---|---|---|
| `GET /api/v1/payments/pricing` | No | GET | Show plan options on paywall screen |
| `GET /api/v1/payments/balance` | Yes | GET | Check paywall, credits, subscription status |
| `POST /api/v1/payments/create-order` | Yes | POST | Start one-time purchase |
| `POST /api/v1/payments/capture-order` | Yes | POST | Complete one-time purchase after PayPal approval |
| `POST /api/v1/payments/create-subscription` | Yes | POST | Start subscription |
| `POST /api/v1/payments/cancel-subscription` | Yes | POST | Cancel active subscription |
| `GET /api/v1/payments/history` | Yes | GET | Show billing history |

---

## Error Responses

| HTTP | Code / Message | Meaning |
|---|---|---|
| `402` | `insufficient_credits` | User has no credits — show paywall |
| `409` | `You already have an active subscription` | Do not allow double subscription |
| `404` | `Pricing plan not found` | Invalid `plan_key` sent |
| `502` | `Payment provider unavailable` | PayPal is down — show retry message |
| `400` | `Payment amount mismatch` | Tampered request — contact support |

---

## What NOT to Do

| Wrong | Why |
|---|---|
| Display `paid_balance` as "Credits" on the home / account screen | It only counts one-off purchases. A subscriber with no bundle buys will see "0 Credits" despite having usable sessions. **Always display `total_usable` instead.** |
| Decide paywall logic on the frontend using session counts | Backend already computes `paywall_required` — use it |
| Call capture-order for subscriptions | Subscriptions have no capture step — webhook handles it |
| Show paywall to users with `subscription.status == "cancelled"` | Cancelled users still have sessions until period ends |
| Block sessions when `total_usable == 0` without checking `paywall_required` | Intro and first foundation are free — `total_usable` can be 0 and still be fine |

---

## Quick Checklist for Frontend Developer

- [ ] Home screen calls `GET /api/v1/payments/balance` on load
- [ ] Show paywall when `paywall_required: true`
- [ ] `GET /payments/pricing` — load plans dynamically, do not hardcode prices or perks
- [ ] Render `perks` array as a bullet list on each plan card (handle empty array gracefully)
- [ ] **Use `flutter_inappwebview` with `shouldOverrideUrlLoading` to intercept `kaya://` deep links** — see [WebView Deep-Link Handling (Flutter)](#webview-deep-link-handling-flutter). Prevents `net::ERR_UNKNOWN_URL_SCHEME`.
- [ ] One-time purchase: create order → open PayPal URL in `PayPalWebView` → capture → refresh balance
- [ ] Subscription: create subscription → open PayPal URL in `PayPalWebView` → wait 3s → refresh balance (no capture)
- [ ] Show `subscription.sessions_remaining` and `subscription.renews_at` when subscribed
- [ ] Handle `subscription.status == "past_due"` with a warning banner
- [ ] Show billing history from `GET /payments/history`
