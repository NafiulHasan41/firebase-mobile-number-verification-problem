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
| `subscription_monthly` | `0` | **`POST /create-subscription`** | ❌ NO — returns `409` if user has an `active`, `past_due`, or `cancelled` (within period) sub |

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
- `POST /create-subscription` — rejects with `409` if the user already has an `active`, `past_due`, or `cancelled` (within period) subscription.

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
| `subscription` | `object \| null` | Subscription details, or `null` if none. Also `null` when the subscription is `expired`. **An `active` subscription is always returned even when `sessions_remaining == 0`** — use that state to show "All sessions used this period — renews on [date]" instead of the paywall. |
| `subscription.status` | `string` | `"active"` \| `"cancelled"` \| `"past_due"` \| `"failed"` — see [Subscription States](#subscription-states). Note: `"expired"` rows are hidden by the backend and return `subscription: null`. |
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
| `sessions_taken` | `int` | Total coaching sessions ever started — **excludes general chat threads** (intro, foundation, follow-up only) |
| `paywall_required` | `bool` | **Show paywall when this is true** — but see note below about `subscription.status == "failed"` |

### Paywall Logic

```dart
final balance = await api.getBalance();
final sub = balance['subscription'];

// Step 1 — failed initial payment (money was never taken — offer retry)
if (sub?['status'] == 'failed') {
  showInitialPaymentFailed();
  return;
}

// Step 2 — paywall gate (total_usable == 0, free sessions consumed)
// Branch on WHY there are no credits — each state needs a different screen.
if (balance['paywall_required'] == true) {
  if (sub?['status'] == 'active') {
    // Active sub, but both paid_balance AND sessions_remaining are 0.
    // Subscription will renew — do NOT show paywall or subscription options.
    showSessionsUsedThisPeriod(renewsAt: sub['renews_at']);
  } else if (sub?['status'] == 'past_due') {
    // Renewal is failing, no sessions left. Need to fix payment — not subscribe again.
    showPastDueNoSessions(); // "Update payment method" — NOT the standard paywall
  } else {
    // No active subscription, no credits — show all purchase options.
    showPaywall();
  }
  return;
}

// Step 3 — user has usable sessions (total_usable > 0)
// total_usable = paid_balance + subscription.sessions_remaining (both pools combined).
// A user with active sub at sessions_remaining=0 but paid_balance=2 lands here — correct.
if (sub?['status'] == 'past_due') {
  showPastDueWarningBanner(); // warn about payment issue but don't block
}
allowSessionStart();
```

`paywall_required` is `true` when ALL of these are true:
1. Intro is completed (`intro_completed: true`)
2. Free foundation slot is used (`foundation_free_used: true`)
3. `total_usable == 0` — **both** `paid_balance` AND `subscription.sessions_remaining` are zero

> **Key point:** `total_usable` is the combined sum of paid (single/bundle) credits and subscription sessions. A user with an active subscription at `sessions_remaining = 0` but `paid_balance = 2` has `total_usable = 2` and `paywall_required = false` — they proceed straight to `allowSessionStart()`. The subscription's session count alone does not determine the paywall.
>
> **`past_due` + no sessions:** the standard paywall must NOT be shown — the user already has a subscription (even if failing). Show "update payment method" instead.

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
  "plan_key": "single_session",
  "plan_label": "Single Session",
  "sessions_credited": 1,
  "new_balance": 1
}
```

| Field | Type | Description |
|---|---|---|
| `status` | `string` | `"completed"` on first successful capture, or `"already_processed"` on a retry of an already-captured order. |
| `plan_key` | `string` | Always present. Which plan was paid for: `"single_session"` or `"bundle_3"`. **Use this to drive the success screen** — see "Showing the right success screen" below. |
| `plan_label` | `string` | Always present. Human-readable label (e.g. `"Single Session"`, `"3-Session Bundle"`). Safe to display directly. |
| `sessions_credited` | `int` | **Only present when `status == "completed"`** (first capture). Absent on `"already_processed"` — use `new_balance` instead. |
| `new_balance` | `int` | Always present. The user's `paid_balance` after this capture. |

After this, `paid_balance` on `/payments/balance` will increase by the sessions purchased. Safe to call multiple times — idempotent.

#### Showing the right success screen

After PayPal redirects to `kaya://payment/success`, the WebView closes and the app's state is fresh — Flutter no longer remembers which plan the user originally tapped. Use the `plan_key` from the capture response to branch the success UI:

```dart
final capture = await api.post(
  '/api/v1/payments/capture-order',
  body: {'order_id': orderId},
);

switch (capture['plan_key']) {
  case 'single_session':
    showSuccess('Single session purchased — 1 session ready to use.');
  case 'bundle_3':
    showSuccess('${capture['plan_label']} purchased — '
                '${capture['sessions_credited']} sessions added.');
  default:
    showSuccess('Purchased — ${capture['sessions_credited']} sessions added.');
}
```

Don't try to remember the selected plan in app state across the WebView round-trip — the deep-link redirect can rebuild the route stack. The capture response is the source of truth.

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
      // sessions_credited is absent on already_processed retries — use new_balance as fallback
      final credited = capture['sessions_credited'] ?? capture['new_balance'];
      showSnackbar('Purchased! $credited sessions added.');
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
| `"cancelled"` | User cancelled (or PayPal cancelled after failed retries), still has sessions until period ends | "Cancelled — sessions available until [renews_at]" | ❌ No |
| `"past_due"` | Renewal charge failed, PayPal retrying | Warning banner: "Payment failed — update your payment method". Sessions still usable during retries | ✅ Yes (and "Update payment method" link) |
| `"failed"` | **First** charge failed on a brand-new subscription — money was never taken | Error: "Initial payment failed — try again with a different card". Offer retry by creating a new subscription | ❌ No (and offer "Try again") |
| `"expired"` | Cancelled subscription whose `current_period_end` has passed — sessions zeroed by backend sweep | Treat identically to `null` — show purchase options. `/balance` **never** returns this status (expired rows are hidden); you will see `subscription: null` instead. Handle defensively in case it appears in other contexts. | ❌ No |
| `"active"` (sessions_remaining == 0) | All sessions used this billing period — subscription is still running and will renew | "All sessions used — renews on [renews_at]". **Do not show the paywall or subscription options.** | ✅ Yes |
| `null` (no subscription) | No active subscription | Show purchase options | — |

### `past_due` → `cancelled` transition

When a renewal charge fails, PayPal marks the subscription `past_due` and retries the charge automatically (~1–3 retries over ~5 days). Two outcomes:

- **Retries succeed** → PayPal fires a renewal webhook → backend resets `sessions_remaining` and sets `status = 'active'` again. No action needed from Flutter.
- **All retries fail** → PayPal cancels the subscription → backend sets `status = 'cancelled'`. The user keeps whatever sessions they had from the last successful billing period until `current_period_end`. After that, a backend sweep (runs every 5 minutes) sets `status = 'expired'` and `sessions_remaining = 0`.

Flutter does not need to handle this transition manually — just re-read `/balance` and render whatever status comes back.

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
| `409` | `You already have an active subscription.` | User has an `active` sub — do not allow a second subscription |
| `409` | `Your current subscription has a failed payment. Update your payment method or cancel it before starting a new one.` | User has a `past_due` sub — prompt them to fix payment or cancel before resubscribing |
| `409` | `sessions_remaining_on_cancelled_sub` | User cancelled but still has sessions within the paid period. Response body also includes `sessions_remaining` (int) and `available_until` (ISO timestamp) — show "You still have N sessions available until [date]" |
| `404` | `No active subscription found.` | Returned by `/cancel-subscription` — no `active` or `past_due` subscription exists for this user |
| `404` | `Pricing plan not found` | Invalid `plan_key` sent to `/create-order` or `/create-subscription` |
| `403` | `This order does not belong to your account.` | Returned by `/capture-order` — the order_id was created by a different user |
| `503` | `Subscription plan not configured yet. Contact support.` | Returned by `/create-subscription` — admin has not set the PayPal plan ID for this plan in the backend config. Report to backend. |
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
| Show paywall to users with `subscription.status == "past_due"` | `past_due` users still have sessions — PayPal is retrying the charge. Show a warning banner, not a paywall. |
| Treat `subscription.status == "expired"` as an unknown/error state | `expired` means the cancelled period ended and sessions were zeroed. Treat it the same as `null` — show purchase options. `/balance` hides expired rows so in practice you will see `subscription: null`. |
| Assume `subscription: null` means "no subscription ever" | It also means the user's active subscription has `sessions_remaining == 0` (all used this period). In that case `paywall_required` will be `true` — rely on that field, not the presence of the subscription object. |
| Show the standard paywall when `subscription.status == "failed"` | A failed sub also produces `paywall_required: true`, but the correct screen is "initial payment failed — try again", not the paywall. Check subscription status first. |
| Show the paywall when `subscription.status == "active"` and `sessions_remaining == 0` | The user still has an active subscription — they've just used all sessions this period. Showing purchase options here causes a confusing 409 when they try to re-subscribe. Show "renews on [date]" instead. |
| Block sessions when `total_usable == 0` without checking `paywall_required` | Intro and first foundation are free — `total_usable` can be 0 and still be fine |

---

## Quick Checklist for Frontend Developer

- [ ] Home screen calls `GET /api/v1/payments/balance` on load
- [ ] Check `subscription.status == "failed"` BEFORE acting on `paywall_required` — show "initial payment failed" screen, not the paywall
- [ ] Check `subscription.status == "active" && sessions_remaining == 0` BEFORE acting on `paywall_required` — show "renews on [date]", never show the paywall or subscription options in this state
- [ ] Show paywall when `paywall_required: true` (and subscription status is not "failed")
- [ ] `GET /payments/pricing` — load plans dynamically, do not hardcode prices or perks
- [ ] Render `perks` array as a bullet list on each plan card (handle empty array gracefully)
- [ ] **Use `flutter_inappwebview` with `shouldOverrideUrlLoading` to intercept `kaya://` deep links** — see [WebView Deep-Link Handling (Flutter)](#webview-deep-link-handling-flutter). Prevents `net::ERR_UNKNOWN_URL_SCHEME`.
- [ ] One-time purchase: create order → open PayPal URL in `PayPalWebView` → capture → refresh balance
- [ ] Subscription: create subscription → open PayPal URL in `PayPalWebView` → wait 3s → refresh balance (no capture)
- [ ] Show `subscription.sessions_remaining` and `subscription.renews_at` when subscribed
- [ ] Handle `subscription.status == "past_due"` with a warning banner (sessions still usable — do NOT show paywall)
- [ ] Handle `subscription.status == "expired"` the same as `null` — show purchase options
- [ ] Handle `past_due` → `cancelled` transition: if status flips to `cancelled` after being `past_due`, show "Subscription ended — sessions available until [renews_at]"
- [ ] Show billing history from `GET /payments/history`

---

## Confirming Subscription Activation After Redirect

Subscriptions don't have a `/capture-subscription` endpoint that mirrors `/capture-order`. After the user approves on PayPal and the WebView returns `PayPalSuccess`, the backend doesn't yet know the sub is active — PayPal sends a `BILLING.SUBSCRIPTION.ACTIVATED` webhook that takes ~2-5 seconds to arrive. The frontend confirms activation by polling `GET /payments/balance` and reading the `subscription` object.

### How to identify which subscription just succeeded

`/payments/balance` returns:

```json
{
  "subscription": {
    "status": "active",
    "plan_key": "subscription_monthly",
    "sessions_remaining": 3,
    "renews_at": "2026-05-30T10:00:00+00:00"
  },
  "total_usable": 3,
  ...
}
```

**`subscription.plan_key`** is the equivalent of the `plan_key` returned by `/capture-order` for one-time purchases — use it to drive the success screen. **`subscription.status`** tells you whether the activation actually completed.

### Recommended pattern — poll until active

The `subscribe()` example earlier shows a single 3-second wait, which is fine for the happy path. For more robust handling, poll up to a few times in case the webhook is slow:

```dart
Future<void> subscribe() async {
  final approvalResp = await api.post(
    '/api/v1/payments/create-subscription',
    body: {'plan_key': 'subscription_monthly'},
  );

  final result = await Navigator.of(context).push<PayPalResult>(
    MaterialPageRoute(
      builder: (_) => PayPalWebView(approvalUrl: approvalResp['approval_url']),
    ),
  );

  if (result is! PayPalSuccess) {
    showSnackbar('Subscription not completed.');
    return;
  }

  // Poll /balance up to 4 times (waiting 2s between each) until the
  // BILLING.SUBSCRIPTION.ACTIVATED webhook lands and the row appears.
  Map<String, dynamic>? sub;
  for (int i = 0; i < 4; i++) {
    await Future.delayed(const Duration(seconds: 2));
    final balance = await api.get('/api/v1/payments/balance');
    sub = balance['subscription'];
    if (sub != null && sub['status'] == 'active') break;
  }

  // Now branch on what we got
  if (sub == null) {
    // Webhook still hasn't arrived after ~8s — show a soft-pending UI.
    showSnackbar('Almost there — your subscription will appear shortly.');
    return;
  }

  switch (sub['status']) {
    case 'active':
      showSuccess(
        'Monthly plan activated — ${sub['sessions_remaining']} '
        'sessions ready for this month.',
      );
    case 'failed':
      // First charge bounced — money was never taken. Offer retry.
      showError(
        'Your subscription couldn\'t start — the payment was declined. '
        'Try again with a different method.',
      );
    case 'past_due':
      // Rare on first activation, but possible. Same retry path.
      showError(
        'Payment didn\'t go through. Please update your payment method.',
      );
    case 'cancelled':
      // PayPal cancelled immediately after activation — extremely rare.
      // User keeps any sessions until current_period_end.
      showSnackbar('Subscription cancelled. Any sessions remain until the period ends.');
    case 'expired':
      // Period already ended — treat same as no subscription.
      showSnackbar('Subscription expired. Please subscribe again.');
    default:
      showSnackbar('Subscription state: ${sub['status']}');
  }

  await refreshBalance();
}
```

### Status meaning right after the redirect

| `subscription.status` | Likely cause | What to do |
|---|---|---|
| `"active"` | Happy path — webhook arrived, session credit landed. | Show "Monthly plan activated" success screen using `plan_key` + `sessions_remaining`. |
| `"failed"` | First charge bounced (declined card, insufficient funds, etc.). PayPal will NOT retry; this subscription is dead. | Show an error and offer to start a fresh `/create-subscription` attempt with a different payment method. |
| `"past_due"` | Very rare immediately after first approval (would mean the activation succeeded then a renewal failed before the user reached this screen). | Same handling as the `past_due` UI elsewhere — banner asking the user to update payment method. |
| `"cancelled"` | PayPal cancelled immediately after activation (extremely rare). | Show "Subscription cancelled — any sessions remain until period ends." Treat like the normal cancelled state. |
| `"expired"` | Period already ended — extremely rare to see this right after activation. | Treat same as `null` — show purchase options. |
| `null` / no `subscription` field | Webhook hasn't arrived yet. | Show a soft "almost there" message and let `/balance` refresh again on the next screen load. The user will also receive the push notification when the webhook lands. |

### Why this is different from one-time purchases

- **One-time:** The frontend calls `/capture-order` synchronously and the response is the source of truth. First capture returns `plan_key`, `plan_label`, `sessions_credited`, and `new_balance`. Retries (`already_processed`) return `plan_key`, `plan_label`, and `new_balance` — `sessions_credited` is absent.
- **Subscription:** Activation is asynchronous via webhook. The frontend can't get a synchronous confirmation — it must poll `/balance` and inspect `subscription.status` and `subscription.plan_key`.

### Push notification on activation

In addition to what the frontend polls, the backend sends a push notification when the subscription activation webhook fires:

> **Title:** Kaya
> **Body:** Monthly plan activated — 3 sessions ready for this month.
> **type:** `payment_success`
> **data.action:** `open_balance`

This is logged to the in-app bell drawer too. So even if the user backgrounds the app before the polling loop finishes, they'll still see the activation confirmation. See the **Notifications Flutter Guide** for handling these.

### What NOT to do

| Wrong | Why |
|---|---|
| Treat `PayPalSuccess` from the WebView as confirmation that the subscription is active | The user only *approved* on PayPal — the activation depends on the webhook landing. Always read `/balance` to confirm `status == "active"`. |
| Hardcode "Monthly subscription activated" without checking `subscription.status` | If the first charge fails, status will be `"failed"` and you'll lie to the user. |
| Wait forever in a polling loop | Cap the polling (4 tries × 2s is plenty). If the webhook still hasn't arrived, surface a soft-pending UI — the next `/balance` refresh will catch up. |
| Use `subscription_id` from the `/create-subscription` response as your primary identifier | That's the PayPal-side ID. For UI purposes, `subscription.plan_key` from `/balance` is what you want. |
