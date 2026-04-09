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
| `paid_balance` | `int` | Number of individually purchased session credits remaining |
| `subscription` | `object \| null` | Active subscription details, or null if none |
| `subscription.status` | `string` | `"active"` \| `"cancelled"` \| `"past_due"` |
| `subscription.plan_key` | `string` | e.g. `"subscription_monthly"` |
| `subscription.sessions_remaining` | `int` | Sessions left in current billing period |
| `subscription.renews_at` | `string \| null` | ISO timestamp of next renewal |
| `free_sessions.intro_completed` | `bool` | Whether intro was properly completed with consent |
| `free_sessions.foundation_free_used` | `bool` | Whether the free foundation slot has been consumed |
| `total_usable` | `int` | Total sessions the user can start right now (paid_balance + subscription sessions) |
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

Open `approval_url` in an in-app browser or WebView. PayPal redirects to `kaya://payment/success` on approval or `kaya://payment/cancelled` on cancel.

```dart
// Handle the deep link return
if (uri.scheme == 'kaya' && uri.host == 'payment') {
  if (uri.path == '/success') {
    await captureOrder(orderId);
  } else if (uri.path == '/cancelled') {
    showCancelledMessage();
  }
}
```

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

Same deep-link handling as one-time purchase. After approval, PayPal fires a webhook to the backend which automatically credits the sessions — no capture step needed.

```dart
// After returning from PayPal approval
if (uri.path == '/success') {
  // No capture needed for subscriptions
  // Re-fetch balance to confirm sessions are credited
  await refreshBalance();
}
```

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

## Subscription States

| `subscription.status` | Meaning | What to show |
|---|---|---|
| `"active"` | Subscription running, sessions available | Show sessions remaining + renewal date |
| `"cancelled"` | User cancelled, still has sessions until period ends | Show "cancelled — sessions available until [date]" |
| `"past_due"` | Renewal payment failed | Show warning — sessions may be cut off |
| `null` (no subscription) | No active subscription | Show purchase options |

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

Returns last 20 payments. Use this for a billing history screen.

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
- [ ] One-time purchase: create order → open PayPal URL → capture → refresh balance
- [ ] Subscription: create subscription → open PayPal URL → refresh balance (no capture)
- [ ] Show `subscription.sessions_remaining` and `subscription.renews_at` when subscribed
- [ ] Handle `subscription.status == "past_due"` with a warning banner
- [ ] Show billing history from `GET /payments/history`
