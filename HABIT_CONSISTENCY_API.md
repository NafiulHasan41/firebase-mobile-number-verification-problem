# Habit Consistency Score — API Guide

For the **Consistency card** on the Home Screen. Auto-computed from the user's
habit completions for the current week — no user input required.

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_identifier>
Content-Type: application/json
```

---

## Get Weekly Consistency Score

```
GET /api/v1/me/habits/consistency
```

No request body. No query parameters.

---

### Example Response — user has habits and completed some

```json
{
  "score": 75,
  "done": 3,
  "expected": 4,
  "period": "week",
  "week_start": "2026-04-20",
  "week_end": "2026-04-26",
  "message": "Keep up the great work! You've been consistent with your habits this week."
}
```

### Example Response — perfect week

```json
{
  "score": 100,
  "done": 4,
  "expected": 4,
  "period": "week",
  "week_start": "2026-04-20",
  "week_end": "2026-04-26",
  "message": "Perfect week! You've completed every habit."
}
```

### Example Response — no habits added yet

```json
{
  "score": 100,
  "done": 0,
  "expected": 0,
  "period": "week",
  "week_start": "2026-04-20",
  "week_end": "2026-04-26",
  "message": "Add your first habit to start tracking consistency."
}
```

---

### Field Reference

| Field | Type | Description |
|---|---|---|
| `score` | integer | Consistency score 0–100. Use this to fill the circular ring indicator |
| `done` | integer | Number of habit completions logged this week |
| `expected` | integer | Number of habit completions expected up to today |
| `period` | string | Always `"week"` — current ISO week (Mon–Sun) |
| `week_start` | string | Monday of the current week in `YYYY-MM-DD` |
| `week_end` | string | Sunday of the current week in `YYYY-MM-DD` |
| `message` | string | Dynamic motivational message — display directly in the card |

---

### Score Calculation (handled by backend)

```
score = (done / expected) × 100   →  rounded, capped at 100
```

Expected completions are calculated per habit frequency up to **today**:

| Frequency | Expected per week (up to today) |
|---|---|
| `daily` | Number of days elapsed since Monday (1–7) |
| `every_other_day` | `ceil(days_elapsed / 2)` |
| `weekly` | `1` |
| `bi_weekly` | `1` if full week elapsed, else `0` |
| `monthly` | Not counted (skipped) |

The backend always uses the **server date** — the client does not need to send any date.

---

### Message by Score Range

| Score | Message |
|---|---|
| `100` | `"Perfect week! You've completed every habit."` |
| `75 – 99` | `"Keep up the great work! You've been consistent with your habits this week."` |
| `50 – 74` | `"Good progress — keep building that momentum."` |
| `0 – 49` | `"Every step counts. You've got this!"` |
| No habits | `"Add your first habit to start tracking consistency."` |

---

### Error Responses

| Status | Reason |
|---|---|
| `401` | Missing or invalid Firebase token |
| `403` | Session active on another device |

---

## Home Screen Widget Logic

### Step 1 — Load on Home Screen mount

```
Call GET /api/v1/me/habits/consistency

If 200 → render the Consistency card
If 401/403 → handle auth error
```

---

### Step 2 — Render the card

```
Show:  label "Consistency" + chart icon
Show:  response.message  (motivational text)
Show:  circular ring filled to response.score %
Show:  response.score as the number inside the ring
Show:  "Score" label below the number
```

---

### Step 3 — Circular ring fill

```dart
// score is 0–100
double fillPercent = response.score / 100.0;

// Example with CircularProgressIndicator
CircularProgressIndicator(
  value: fillPercent,     // 0.0 → empty, 1.0 → full ring
  ...
)
```

Ring fill examples:
- `score: 100` → full ring (complete circle)
- `score: 75`  → 75% filled (gap at top-right)
- `score: 0`   → empty ring

---

### Step 4 — When to refresh

```
Refresh on every Home Screen mount (app open / tab switch)
No need to poll — score only changes when a habit is logged
```

---

## Tested Live Result (2026-04-23)

User: NAFIUL (`+8801305727216`)
Habit: 1 active daily habit, completed once this week

```json
{
  "score": 100,
  "done": 1,
  "expected": 1,
  "period": "week",
  "week_start": "2026-04-20",
  "week_end": "2026-04-26",
  "message": "Perfect week! You've completed every habit."
}
```

---

## Summary

| Endpoint | Auth | Purpose |
|---|---|---|
| `GET /api/v1/me/habits/consistency` | Firebase token | Weekly habit consistency score for the Home Screen card |

**This endpoint is read-only.** The score updates automatically as the user logs habits via `POST /api/v1/me/habits/{id}/log`.
