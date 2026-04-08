# Kaya Frontend — Update Guide
_For the Flutter developer. Backend changes that require frontend updates._
_Date: 2026-04-08_

---

## What Changed (Summary)

| # | Change | Impact |
|---|--------|--------|
| 1 | `POST /api/v1/sessions/start` now returns the coach's first message | Frontend must display it — never show a blank chat screen |
| 2 | Intro sessions have no expiry | No countdown timer for intro — hide it entirely |
| 3 | Coach always knows user's name | No change needed — backend handles this |

---

## Change 1 — `POST /api/v1/sessions/start` Returns First Message

### What changed

Previously: `POST /sessions/start` returned only a session ID. Frontend would show a blank chat screen and wait for the user to type first.

Now: The coach's opening message is included directly in the `/start` response. The user arrives to a live conversation, not a blank screen.

### Updated response shape

```json
POST /api/v1/sessions/start
{ "type": "intro" }

→ 200 OK

{
  "session_id": "1909d362-91a0-4ee1-a003-e37ecc2ebde3",
  "type": "intro",
  "status": "active",
  "time_remaining_seconds": null,
  "expires_at": null,
  "first_message": "Hey Nafiul! Welcome — I'm really glad you're here. I'm Kaya, your AI Functional Medicine Health Coach...",
  "first_message_type": "text"
}
```

New fields:

| Field | Type | Description |
|-------|------|-------------|
| `first_message` | `string \| null` | Coach's opening message. Null only if something went wrong (rare). |
| `first_message_type` | `string \| null` | `"text"` or `"options"`. Same as `message_type` on a normal `done` event. |

### What to do

**On `POST /sessions/start` success:**
1. Take the `session_id` from the response.
2. If `first_message` is not null, immediately add it to the message list as an assistant bubble — do **not** send any message to get the opener.
3. Enable the input field so the user can respond.

```dart
final result = await api.startSession(type: 'intro');
final sessionId = result['session_id'];

setState(() {
  // Show the coach's opening immediately
  if (result['first_message'] != null) {
    messageList.add(Message(
      role: 'assistant',
      content: result['first_message'],
      messageType: result['first_message_type'] ?? 'text',
    ));
  }
  sendButtonEnabled = true;
});
scrollToBottom();
```

### Loading history for an existing session

If the user leaves and comes back to a session that was already started, use `GET /api/v1/sessions/{id}/messages` as before. The coach's opening message is already in the history — you do not need to handle it separately. Just render the full history list as normal.

```
GET /api/v1/sessions/{session_id}/messages
```

The first item in `messages` will be the coach's opening (role: `assistant`). Render it like any other assistant bubble.

### What NOT to do

| Wrong | Why |
|-------|-----|
| Show a blank chat and wait for user to type first | User has no idea what to do |
| Send a "Hi" message to trigger the opener | Opening is already done — this creates a duplicate |
| Ignore `first_message` and re-fetch history | Unnecessary request; history already contains the opening |

---

## Change 2 — Intro Sessions Have No Expiry

### What changed

Foundation and follow-up sessions expire after 24 hours. The intro session now has **no expiry** — the user can take as long as they need, leave, come back days later, and continue.

### How to detect: check the type

When `session.type == "intro"`, treat it as open-ended. When `session.type == "foundation"` or `"followup"`, use the timer as before.

### Updated `GET /sessions/active` response for intro

```json
{
  "active_session": {
    "session_id": "1909d362-...",
    "type": "intro",
    "status": "active",
    "time_remaining_seconds": null,
    "expires_at": null,
    "progress": { ... },
    "started_at": "2026-04-08T10:00:00Z"
  }
}
```

`time_remaining_seconds` and `expires_at` are **always null for intro** — this is correct, not an error.

### Updated `POST /sessions/start` response for intro

Same — `time_remaining_seconds: null`, `expires_at: null` for intro.

For foundation and follow-up, these fields are unchanged — they still return the numeric countdown and the expiry timestamp.

### What to do

**Hide the countdown timer entirely for intro sessions.**

```dart
// Show timer only for foundation/followup — not for intro
if (session.type != 'intro' && session.timeRemainingSeconds != null) {
  showCountdownTimer(session.timeRemainingSeconds);
}
```

**Do not show any expiry warning for intro.** Do not show "Session expires in X hours". The intro has no time limit.

**Foundation and follow-up are unchanged** — keep the countdown timer as-is for those.

### Intro ends in only two ways

1. User gives consent → agent calls `end_session` → `message_type: session_ended` arrives in the stream → handle as normal session end.
2. User taps "Leave session" (force end) → same `session_ended` handling.

There is no timer-based expiry for intro. If you were showing a warning like "Your session expires in 3 hours" in the intro, remove it.

---

## No Changes Required For

| Topic | Reason |
|-------|--------|
| Streaming endpoints | Unchanged — same SSE events, same format |
| Chat threads | Unchanged |
| Session message endpoints | Unchanged |
| Session history (`GET /sessions/{id}/messages`) | Unchanged — opener is just the first message |
| Session feedback | Unchanged |
| Auth flow | Unchanged |
| Payment flow | Unchanged |
| Options buttons (`message_type: options`) | Unchanged |
| Session ended handling (`message_type: session_ended`) | Unchanged |

---

## Quick Checklist for Frontend Developer

- [ ] `POST /sessions/start` — read `first_message` and display it immediately as an assistant bubble
- [ ] `POST /sessions/start` — do not show a blank screen or send a trigger message
- [ ] Intro session — hide countdown timer when `session.type == "intro"`
- [ ] Intro session — do not show expiry warning when `session.type == "intro"`
- [ ] Foundation / follow-up — timer and expiry behavior unchanged
- [ ] Session history (`GET /sessions/{id}/messages`) — no change, render as before
