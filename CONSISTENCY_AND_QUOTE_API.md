# Consistency Rating & Quote of the Day — API Guide

All mobile app endpoints require authentication.

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_identifier>
Content-Type: application/json
```

---

## 1. Consistency / Accountability Rating

A daily self-check widget on the Home Screen. After the user completes a foundation session, they can rate how consistent they have been with their agreed plan on a scale of 1–10.

**Only available after a foundation session is completed.** Returns `403` before that.

---

### Submit Today's Rating

```
POST /api/v1/me/consistency
```

**Request Body**

| Field | Type | Required | Description |
|---|---|---|---|
| `rating` | integer | Yes | 1–10. How consistent the user has been today |
| `note` | string | No | Optional note. Default: `""` |

**Example**
```json
POST /api/v1/me/consistency

{
  "rating": 7,
  "note": "Stuck to my sleep and nutrition plan today"
}
```

**Success Response** `200 OK`
```json
{
  "status": "ok",
  "action": "created",
  "rating": 7,
  "scale": 10,
  "percentage": 70
}
```

**`action` field**

| Value | Meaning |
|---|---|
| `"created"` | First submission for today — new entry added |
| `"updated"` | User already rated today — existing entry overwritten |

Use this to show different UI feedback:
- `"created"` → show success animation / "Rating saved!"
- `"updated"` → show subtle "Rating updated" message

**Date security — handled by the backend**
- The date is always set to `CURRENT_DATE` on the server — the user **cannot** submit for a past or future date
- One entry per user per day enforced via DB unique constraint `(user_id, date)`
- Submitting again on the same day always overwrites — no duplicate errors

**Error Responses**

| Status | Reason |
|---|---|
| `403` | Foundation session not yet completed — hide this widget |
| `422` | Rating is not between 1 and 10 |

---

### Get Consistency History

```
GET /api/v1/me/consistency?limit=30
```

**Query Parameters**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | 30 | Number of past entries to return |

**Example Response**
```json
{
  "entries": [
    { "rating": 7, "note": "Good day", "date": "2026-04-06" },
    { "rating": 5, "note": "", "date": "2026-04-05" },
    { "rating": 8, "note": "Best week so far", "date": "2026-04-04" }
  ],
  "latest": 7,
  "latest_pct": 70,
  "previous": 5,
  "previous_pct": 50,
  "average": 6.7,
  "average_pct": 67,
  "trend": "up",
  "trend_delta": 2,
  "scale": 10,
  "count": 3
}
```

**Field Reference**

| Field | Description |
|---|---|
| `latest` | Today's (or most recent) rating |
| `latest_pct` | `latest / 10 * 100` — use for progress bar fill |
| `previous` | The rating before the latest one |
| `previous_pct` | Previous as percentage |
| `average` | Rolling average across all returned entries |
| `average_pct` | Average as percentage |
| `trend` | `"up"` / `"down"` / `"stable"` — show arrow indicator |
| `trend_delta` | Numeric difference (`latest - previous`) e.g. `+2` or `-1` |
| `scale` | Always `10` — the denominator |
| `count` | Number of entries returned |

**When `entries` is empty:** user has not submitted any ratings yet — show the input widget but no history.

---

### Home Screen Widget Logic

**Step 1 — Load the widget on Home Screen mount**

```
Call GET /api/v1/me/consistency

If response is 403:
  → Hide the widget completely (foundation not done yet)
  → Do not show it until a foundation session is completed

If response is 200:
  → Proceed to Step 2
```

---

**Step 2 — Determine widget state**

Check whether the user already submitted a rating today:

```
today = current date in "YYYY-MM-DD" format

alreadyRatedToday = (entries.length > 0 && entries[0].date === today)
```

There are three possible states:

| State | Condition | What to show |
|---|---|---|
| **Never rated** | `count === 0` | Input prompt only, no history |
| **Rated today** | `alreadyRatedToday === true` | Today's rating + allow editing |
| **Rated before, not today** | `count > 0 && !alreadyRatedToday` | Input prompt + yesterday's history |

---

**Step 3 — Render based on state**

```
If count === 0:
  Show: "How consistent have you been today? (1–10)"
  Show: Rating input (1–10 tap bar)
  Hide: Progress bar and trend (no data yet)

If alreadyRatedToday === true:
  Show: Progress bar filled to latest_pct
  Show: "You rated {latest}/10 today ({latest_pct}%)"
  Show: Trend arrow + trend_delta vs yesterday
       "up"     → ↑ green  e.g. "+2 since yesterday"
       "down"   → ↓ red    e.g. "-1 since yesterday"
       "stable" → → gray   "Same as yesterday"
  Show: Average label "Avg: {average}/10"
  Show: Edit button or tappable rating bar to update

If count > 0 && !alreadyRatedToday:
  Show: "How consistent have you been today? (1–10)"
  Show: Rating input (1–10 tap bar)
  Show: "Yesterday: {previous}/10" as a subtle reference
  Show: Average label "Avg: {average}/10"
```

---

**Step 4 — User submits or updates a rating**

```
User taps a number (1–10)
  → Call POST /me/consistency { "rating": selectedNumber }
  → Show loading spinner on the widget

On success:
  → If action === "created" → show "Rating saved!" feedback
  → If action === "updated" → show "Rating updated" feedback
  → Call GET /api/v1/me/consistency again to refresh
  → Widget re-renders (now alreadyRatedToday = true)

On error:
  → Show error message inline on the widget
  → Do not clear the user's selection
```

---

**Step 5 — Handling skipped days**

If the user did not rate yesterday (or skipped multiple days), the history will have gaps. This is normal and expected — do not try to fill gaps or backfill.

```
Example entries array with a gap:
[
  { "rating": 7, "date": "2026-04-06" },  ← today (submitted)
  { "rating": 5, "date": "2026-04-04" },  ← 2 days ago (Apr 5 was skipped)
  { "rating": 8, "date": "2026-04-02" },  ← skipped Apr 3
]

alreadyRatedToday = entries[0].date === today  → true
trend = "up" (7 vs 5, the two most recent entries)
trend_delta = +2
```

Rules for skipped days:
- `trend` and `trend_delta` always compare the **two most recent entries** regardless of how many days apart they are — backend handles this
- Do not show "you missed X days" messaging — just show the prompt naturally the next time they open the app
- The widget always shows the input prompt if `!alreadyRatedToday`, even after a long gap

---

**Full flow diagram**

```
App opens / Home Screen loads
        │
        ▼
GET /api/v1/me/consistency
        │
   ┌────┴────┐
  403       200
   │         │
Hide       Check entries[0].date === today?
widget          │
           ┌───┴───┐
          YES      NO
           │        │
     Show today's  Show input
     rating +      prompt +
     trend         past average
           │        │
           └───┬────┘
               │
         User taps rating
               │
               ▼
     POST /api/v1/me/consistency
      { "rating": N }
               │
               ▼
     Refresh GET /api/v1/me/consistency
               │
               ▼
     Widget updates (rated today = true)
```

---

---

## 2. Quote of the Day

A content card on the Home Screen. Admin schedules quotes from the dashboard and they appear for all users on that date.

---

### Get Today's Quote

```
GET /api/v1/me/quote
```

No request body. No query parameters.

**Example Response — quote exists**
```json
{
  "quote": {
    "id": "a1b2c3d4-...",
    "text": "The greatest wealth is health.",
    "author": "Virgil",
    "category": "health",
    "date": "2026-04-06"
  }
}
```

**Example Response — no quote available**
```json
{
  "quote": null
}
```

**Logic (handled by backend, no action needed on frontend):**
1. If a quote is scheduled for today → return it
2. If no quote is scheduled → return a random active quote from the pool
3. If no quotes exist at all → return `null`

**Field Reference**

| Field | Description |
|---|---|
| `text` | The quote or health fact text |
| `author` | Author name — may be `null` or empty string if not set |
| `category` | One of: `general`, `health`, `nutrition`, `movement`, `sleep`, `stress`, `mindfulness`, `motivation` |
| `date` | The scheduled date if pinned, or `null` if from random pool |

---

### Home Screen Widget Logic

```
1. Call GET /me/quote on every app open / home screen load
2. If quote is null → hide the section entirely (or show placeholder)
3. If quote exists:
   - Show quote.text in italic
   - If quote.author is not empty → show "— {author}" below
   - Optionally show category badge
4. Refresh daily (on app foreground or home screen mount)
```

---

## Summary Table

| Endpoint | Who Calls It | Auth | Purpose |
|---|---|---|---|
| `POST /me/consistency` | Mobile app | Firebase token | Submit today's consistency rating |
| `GET /me/consistency` | Mobile app | Firebase token | Get history + trend for Home Screen widget |
| `GET /me/quote` | Mobile app | Firebase token | Get today's quote for Home Screen |
| `GET /admin/quotes` | Admin dashboard | Admin JWT | List all quotes |
| `POST /admin/quotes` | Admin dashboard | Admin JWT | Create a new quote |
| `PUT /admin/quotes/{id}` | Admin dashboard | Admin JWT | Edit quote or toggle active |
| `DELETE /admin/quotes/{id}` | Admin dashboard | Admin JWT | Delete a quote |

---

## Admin Quote Rules

- **One quote per date** — trying to schedule two quotes on the same date returns `409` with message: *"A quote is already scheduled for YYYY-MM-DD. Edit or delete it first."*
- **Unscheduled quotes** (`scheduled_date = null`) go into the random pool — shown on days with no scheduled quote
- **`is_active = false`** quotes are never shown to users, even if scheduled for today
- Admin can toggle active/inactive without deleting
