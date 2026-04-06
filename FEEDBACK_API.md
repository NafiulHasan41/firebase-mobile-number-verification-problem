# Feedback API — Usage Guide

All endpoints require authentication.

```
Authorization: Bearer <firebase_id_token>
X-Device-Id: <device_identifier>
Content-Type: application/json
```

---

## 1. Session Feedback

Submit feedback after a coaching session ends. The form has three fields: rating, happy, and an optional comment.

**Endpoint**
```
POST /api/v1/sessions/{session_id}/feedback
```

**Path Parameter**
| Parameter | Type | Description |
|---|---|---|
| `session_id` | UUID string | The ID of the completed session |

**Request Body**
| Field | Type | Required | Description |
|---|---|---|---|
| `rating` | integer | Yes | Star rating — `0` to `5`. Use `0` if user skips stars |
| `happy` | boolean | Yes | `true` = happy with session, `false` = not happy |
| `comment` | string | No | Optional text. Default: `""` |

**Example — Full feedback**
```json
POST /api/v1/sessions/32395c12-5c68-438c-a739-d1cff2ae69b7/feedback

{
  "rating": 4,
  "happy": true,
  "comment": "Really helpful session!"
}
```

**Example — No comment, no stars**
```json
{
  "rating": 0,
  "happy": false
}
```

**Success Response** `200 OK`
```json
{
  "status": "ok",
  "feedback_id": "d0a2d077-6f43-47f2-8fe5-c8d1e5bc503d"
}
```

**Error Responses**
| Status | Reason |
|---|---|
| `400` | Rating is not between 0 and 5 |
| `400` | Session is still active — must end first |
| `409` | Already submitted feedback for this session |
| `403` | Session does not belong to this user |
| `404` | Session not found |

**When to call:**
Show the feedback form immediately after a session ends (when session `status` becomes `completed`). One submission per session.

---

## 2. General Feedback

Submit a feedback message, suggestion, or bug report from the app's settings/feedback screen.

**Endpoint**
```
POST /api/v1/me/feedback
```

**Request Body**
| Field | Type | Required | Description |
|---|---|---|---|
| `category` | string | Yes | One of: `"feedback"`, `"suggestion"`, `"bug"` |
| `message` | string | Yes | Feedback text. Min 1 character, max 2000 characters |

**Example — Feedback**
```json
POST /api/v1/me/feedback

{
  "category": "feedback",
  "message": "The session flow feels very natural and easy to follow."
}
```

**Example — Suggestion**
```json
{
  "category": "suggestion",
  "message": "It would be great to have a dark mode option."
}
```

**Example — Bug**
```json
{
  "category": "bug",
  "message": "The app crashes when I try to open past session history."
}
```

**Success Response** `200 OK`
```json
{
  "status": "ok",
  "feedback_id": "71d9a09d-b418-44af-8464-993f12e8ae83"
}
```

**Error Responses**
| Status | Reason |
|---|---|
| `422` | Category is not one of `feedback`, `suggestion`, `bug` |
| `422` | Message is empty or exceeds 2000 characters |

**When to call:**
Triggered from a "Send Feedback" button in settings or profile screen. No limit on how many times a user can submit.

---

## Admin Read Endpoints

Called by the admin dashboard only. Use admin JWT, not a Firebase token.

### Session Feedback List

```
GET /api/v1/admin/feedback/sessions?limit=50&offset=0
```

**Query Parameters**
| Parameter | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | 50 | Number of results |
| `offset` | integer | 0 | Pagination offset |

**Example Response**
```json
{
  "total": 1,
  "happy_count": 1,
  "unhappy_count": 0,
  "positive_pct": 100.0,
  "average_rating": 4.0,
  "feedback": [
    {
      "id": "d0a2d077-6f43-47f2-8fe5-c8d1e5bc503d",
      "session_id": "32395c12-5c68-438c-a739-d1cff2ae69b7",
      "user_id": "98327783-d70f-4734-a918-61d26182402e",
      "name": "Test User",
      "email": "test@example.com",
      "session_type": "intro",
      "happy": true,
      "rating": 4,
      "comment": "Really helpful session!",
      "created_at": "2026-04-06T12:39:41.269724+00:00"
    }
  ]
}
```

`positive_pct` is the percentage of `happy = true` responses. Target from product spec is 80%.

---

### General Feedback List

```
GET /api/v1/admin/feedback/general?limit=50&offset=0&category=bug
```

**Query Parameters**
| Parameter | Type | Default | Description |
|---|---|---|---|
| `limit` | integer | 50 | Number of results |
| `offset` | integer | 0 | Pagination offset |
| `category` | string | *(all)* | Filter by `feedback`, `suggestion`, or `bug` |

**Example Response**
```json
{
  "total": 3,
  "feedback": [
    {
      "id": "71d9a09d-b418-44af-8464-993f12e8ae83",
      "user_id": "98327783-d70f-4734-a918-61d26182402e",
      "name": "Test User",
      "email": "test@example.com",
      "category": "suggestion",
      "message": "Add dark mode please!",
      "created_at": "2026-04-06T17:02:54.228351+00:00"
    }
  ]
}
```

---

## Summary Table

| Endpoint | Who Calls It | Auth | Purpose |
|---|---|---|---|
| `POST /sessions/{id}/feedback` | Mobile app | Firebase token | Session rating + happy/not + comment |
| `POST /me/feedback` | Mobile app | Firebase token | Submit feedback / suggestion / bug |
| `GET /admin/feedback/sessions` | Admin dashboard | Admin JWT | View all session feedback with sentiment |
| `GET /admin/feedback/general` | Admin dashboard | Admin JWT | View feedback / suggestions / bugs |
