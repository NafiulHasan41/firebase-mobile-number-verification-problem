# Edit Profile — API Update

## What Changed

`PUT /api/v1/me/account` and `GET /api/v1/me/account` have been updated to support `date_of_birth` in addition to `name`.

---

## Editable Fields

| Field | Editable | Notes |
|---|---|---|
| `name` | ✅ Yes | Full name, 1–100 characters |
| `date_of_birth` | ✅ Yes | ISO date string `YYYY-MM-DD` |
| `email` | ❌ Read-only | Tied to Firebase auth provider |
| `phone` | ❌ Read-only | Tied to Firebase auth provider |

Email and phone are returned by `GET /account` for display purposes only — they cannot be updated.

---

## GET /api/v1/me/account

Returns `date_of_birth` in the response now.

```
GET /api/v1/me/account
Headers: Authorization, X-Device-Id
```

**Response (200):**
```json
{
  "user_id": "08fb5bd0-614e-4a8f-a9a0-a57795af38b5",
  "name": "Sarah",
  "email": null,
  "phone": "+1234567890",
  "auth_provider": "phone",
  "biometric_enabled": false,
  "data_sharing_consent": false,
  "otp_verified": true,
  "date_of_birth": "1990-05-15",
  "created_at": "2026-04-03T11:15:54.720295+00:00"
}
```

`date_of_birth` is `null` if not set yet.

---

## PUT /api/v1/me/account

Both fields are now **optional** — send only the field(s) you want to update.
At least one field must be provided.

```
PUT /api/v1/me/account
Headers: Authorization, X-Device-Id
Content-Type: application/json
```

### Update name only
```json
{
  "name": "Sarah H."
}
```

### Update date of birth only
```json
{
  "date_of_birth": "1990-05-15"
}
```

### Update both at once
```json
{
  "name": "Sarah H.",
  "date_of_birth": "1990-05-15"
}
```

**Response (200):**
```json
{
  "status": "ok",
  "name": "Sarah H.",
  "date_of_birth": "1990-05-15"
}
```

Fields not included in the request are returned as `null` in the response — this does **not** mean they were cleared. It only reflects what was sent in that request. Use `GET /account` to read the full current state.

**Errors:**
- `422` — Neither `name` nor `date_of_birth` provided
- `422` — `name` is empty string or exceeds 100 characters
- `422` — `date_of_birth` is not a valid date format

---

## Edit Profile Screen — Display Rules

Show fields based on `auth_provider` from `GET /account`:

| `auth_provider` | Show email field | Show phone field |
|---|---|---|
| `google` | ✅ Read-only | ❌ Hide |
| `apple` | ✅ Read-only | ❌ Hide |
| `email` | ✅ Read-only | ❌ Hide |
| `phone` | ❌ Hide | ✅ Read-only |

---

## Important — Frontend State After Save

After a successful `PUT /account`, update the local user state from the response immediately. Do not wait for a re-fetch or navigation — the response already contains the updated values.
